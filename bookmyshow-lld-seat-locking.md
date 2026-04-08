# How BookMyShow Locks a Cinema Seat When 10,000 Users Click It in the Same Millisecond

It's 8 PM. IPL final ticket sale just opened. You and 10,000 other people across India tap the same seat — A4, dead center, fourth row. One millisecond later, exactly **one** person owns that seat. The other 9,999 see "Sorry, this seat is no longer available."

No double bookings. No "two people received tickets for the same seat" disasters. No race conditions visible to the user.

How?

This is the **most-asked LLD interview question** in Indian backend interviews in 2026 — Amazon India, Flipkart, Swiggy, Razorpay all ask some flavor of it. And **80% of candidates botch it** because they go straight to the database and write the wrong UPDATE statement.

This article is the full breakdown: the wrong way, the right way, the edge cases interviewers love to ask about, and the exact two-phase locking pattern that real ticket-booking systems use.

---

## The Problem

You have a `seats` table:

```sql
CREATE TABLE seats (
  seat_id BIGINT PRIMARY KEY,
  show_id BIGINT NOT NULL,
  row CHAR(1),
  col INT,
  status ENUM('available', 'held', 'booked'),
  held_by BIGINT NULL,
  held_until TIMESTAMP NULL,
  booked_by BIGINT NULL
);
```

A user picks seat A4 and clicks "Book". Your API needs to:
1. Make sure no one else can take this seat while the user pays
2. If the user abandons the flow, release the seat automatically
3. If two users somehow click in the same millisecond, only ONE wins
4. If your database server crashes mid-transaction, the seat doesn't get permanently stuck

Sounds simple. It is not.

---

## The Wrong Way (and why every junior writes this)

The naive approach is one SQL statement:

```sql
UPDATE seats
SET status = 'booked', booked_by = :user_id
WHERE seat_id = :seat_id
  AND status = 'available';
```

"Look, the `WHERE status='available'` clause makes it atomic! If two users send this query at the same time, only one update will succeed."

This is **technically correct** — the database guarantees atomicity. But it's still **wrong** for a real ticket-booking flow. Three reasons:

### Problem 1: There's no "holding" state

In a real flow, a user picks seats → goes through payment (could take 2-5 minutes) → confirms. During that whole time, no one else should be able to take those seats. The naive approach locks the seat for **zero seconds** — the moment between SELECT and UPDATE — which doesn't help.

If you UPDATE to `booked` immediately on click and the user abandons, you have a phantom booked seat with no payment. If you DON'T update on click, two users can both reach the payment page for the same seat, and the second one to confirm gets a heart attack.

### Problem 2: Database row locks don't survive process crashes

If you use `SELECT ... FOR UPDATE` to hold the row during payment, you're holding a database transaction open for **minutes**. That's a disaster:
- Connection pool exhaustion (each held transaction = one connection out of action)
- Deadlocks against other queries hitting the same row
- If the API process crashes, the lock is released — but the user thinks they have the seat

### Problem 3: Direct DB hits don't scale to IPL traffic

When IPL final tickets drop, you don't get 10,000 requests per second. You get 100,000 to 1 million requests per second concentrated on a few hundred premium seats. Your database will fall over. You need a **fast in-memory layer** in front.

---

## The Right Way: Two-Phase Locking

The pattern every real ticket-booking system uses:

**Phase 1 — Soft Hold (Redis, fast, in-memory, TTL-based)**
**Phase 2 — Confirmation (Database, slow, durable, transactional)**

The user goes through Phase 1 the moment they click a seat. They reach Phase 2 only when they tap "Pay". If they abandon between the two, Phase 1's TTL expires and the seat becomes available again.

### Phase 1 — Redis Soft Hold

When the user clicks seat A4, your API does ONE Redis command:

```
SET seat:hold:show_42:A4 <user_token> NX EX 600
```

Breaking this down:
- `seat:hold:show_42:A4` — the key, namespaced by show + seat
- `<user_token>` — a random UUID generated for THIS user's hold attempt (we'll see why)
- `NX` — "Only set if the key does not exist." Atomic. If 10,000 users send this command at the same millisecond, exactly ONE Redis instance returns OK; the other 9,999 return nil
- `EX 600` — TTL of 600 seconds (10 minutes). Auto-release if the user abandons

That's it. Phase 1 is one command. It's atomic. It's fast (Redis handles 100K+ ops/sec on a single instance). It auto-releases if anything goes wrong.

```python
def hold_seat(user_id, show_id, seat_id):
    user_token = uuid.uuid4().hex
    key = f"seat:hold:{show_id}:{seat_id}"
    success = redis.set(key, user_token, nx=True, ex=600)
    if success:
        return {"held": True, "token": user_token, "expires_in": 600}
    return {"held": False, "reason": "seat already held"}
```

The `user_token` is the critical detail. We'll need it to safely release the lock later.

### Phase 2 — Database Confirmation

The user pays. Your payment provider returns success. Now you need to convert the soft hold into a permanent booking:

```python
def confirm_booking(user_id, show_id, seat_ids, user_tokens):
    # Sort seat_ids ascending — prevents deadlocks
    pairs = sorted(zip(seat_ids, user_tokens))

    with db.transaction():
        for seat_id, token in pairs:
            # Verify the hold is still valid AND belongs to this user
            cached_token = redis.get(f"seat:hold:{show_id}:{seat_id}")
            if cached_token != token:
                raise Exception(f"Hold expired or stolen for seat {seat_id}")

            # Lock the row in the DB
            row = db.execute(
                "SELECT * FROM seats WHERE seat_id=? AND show_id=? FOR UPDATE",
                (seat_id, show_id)
            ).fetchone()

            if row.status != 'available':
                raise Exception(f"Seat {seat_id} is no longer available")

            db.execute(
                "UPDATE seats SET status='booked', booked_by=? WHERE seat_id=?",
                (user_id, seat_id)
            )

        db.commit()

    # Atomic Redis release using Lua — only delete if token matches
    for seat_id, token in pairs:
        release_lock_safe(f"seat:hold:{show_id}:{seat_id}", token)
```

Two critical details in Phase 2:

**1. Deadlock prevention via ascending lock order.** If user A locks seat 5 then 10, and user B locks seat 10 then 5, the database deadlocks instantly — A waits for B's seat 10, B waits for A's seat 5. The fix is universal: **always lock seats in the same order** (ascending seat ID). Now both users try to lock seat 5 first; whoever gets there first succeeds, the other waits politely.

**2. Token-based Redis release.** When you DELETE the Redis hold, you must verify the token still matches. Why? Imagine this race:

```
T=0:    User A acquires hold on seat A4 (token=abc)
T=600:  User A's hold expires (TTL hit)
T=601:  User B acquires fresh hold on seat A4 (token=xyz)
T=602:  User A's API process finally calls DEL seat:hold:show_42:A4
        → DELETES USER B's hold!
```

To prevent this, the release must be atomic and conditional. Use a Lua script:

```lua
-- release_lock_safe.lua
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
else
    return 0
end
```

This runs as ONE atomic operation in Redis. The DEL only happens if the token matches.

---

## The Clever Part — Why Two Phases Beat One

The genius of separating Phase 1 (Redis) from Phase 2 (DB) is that you get the **best of both worlds**:

| Property | Redis Phase 1 | DB Phase 2 |
|---|---|---|
| Speed | 100K+ ops/sec | ~1K ops/sec |
| Durability | Volatile (acceptable here — we have DB as truth) | Persistent (the source of truth) |
| Auto-release | Built-in via TTL | Manual via timeout job |
| Lock duration | 10 minutes safely | Sub-second per transaction |
| Failure mode | Redis crashes → all holds vanish, seats become bookable | DB crashes → confirmed bookings preserved |

If you tried to do this all in the database, you'd hold transactions open for minutes (catastrophic). If you tried to do it all in Redis, you'd lose confirmed bookings on a Redis restart (also catastrophic). The split lets each layer do what it's good at.

---

## Edge Cases Interviewers Love

### Q1: What if the user closes the browser mid-payment?

The Redis hold's TTL (10 min) expires. The seat becomes available automatically. No cleanup job needed.

### Q2: What if Redis crashes between Phase 1 and Phase 2?

All holds vanish (Redis is in-memory). All the soft-held seats become available again. No double-booking risk because the DB is still the source of truth — confirmed bookings are still in the DB. New users will see the seats as available and try to hold them again. Worst case: a few users see "your hold expired, please retry."

### Q3: What if two users complete payment at exactly the same moment for the same seat?

Impossible — Phase 1 ensures only ONE user can hold the seat. If User A holds it, User B's `SET NX` returns nil and they never reach Phase 2. The race is resolved at Phase 1, not Phase 2.

### Q4: What if the payment provider takes 30 seconds to confirm and the Redis hold expires during that time?

This is the dangerous one. To prevent it:
- Set Redis TTL longer than your max payment timeout (10 min vs 5 min payment SLA gives buffer)
- On confirmation, **re-check the hold** before committing — that's the `cached_token != token` check above
- If the hold expired, fail the booking gracefully and refund the user

Production systems often add a "confirming" intermediate state in the DB to handle this — a row that's neither held nor booked but has a payment in flight.

### Q5: What about Redlock for multi-region Redis?

For most ticket-booking workloads, a single Redis cluster (with Sentinel for HA) is fine. Redlock adds complexity for marginal gains, and Martin Kleppmann famously argued it's not actually safe under network partitions. Stick with single-master Redis unless you have a very specific requirement.

### Q6: What if there's a fencing-token-style requirement?

If the DB confirmation step needs to be safe even against a slow Phase 1 client whose lock expired, use a fencing token: each Redis SET also stores an incrementing counter. The DB UPDATE includes a `WHERE last_held_token = ?` clause. This is overkill for ticket booking but interviewers who've read Kleppmann love it.

---

## Real-World Numbers

- **BookMyShow** handles **50M+ MAU** in India, with traffic spikes during big releases (Pushpa 2, Marvel films, RRR, Salaar) of 10-100x baseline
- **IPL ticket sales**: BookMyShow has been the official ticketing partner for IPL since 2008. A single match (like CSK vs MI in Chennai) sells out in **under 5 minutes** for premium seats — that's millions of users hitting a few hundred seats simultaneously
- **Redis** can handle **100K+ SET NX operations per second** on a single node. A horizontally sharded Redis cluster (sharded by `show_id` hash) can scale this to millions per second
- **Database**: Postgres or MySQL with row-level locking handles **1-5K confirmed bookings per second** per node. With sharding by `show_id`, you can scale this linearly

---

## Key Takeaways

| Lesson | Why |
|---|---|
| **Don't do everything in the database** | DB row locks held for minutes = connection pool death + deadlocks |
| **Two-phase: Redis hold + DB confirm** | Speed + durability + auto-release in one pattern |
| **`SET NX EX` is atomic** | Single Redis command that handles 10K-user races correctly |
| **Token-based release via Lua** | Prevents accidentally deleting another user's lock after TTL expiry |
| **Lock seats in ascending ID order** | Universal deadlock prevention rule for any multi-row transaction |
| **TTL > payment SLA** | Always 2x buffer to avoid expired holds during slow payments |
| **Re-verify hold at confirmation time** | The "is my hold still valid?" check is the safety net for slow flows |
| **Single-master Redis is fine for this** | Redlock is overkill; use Sentinel for HA |

---

## References

1. [Redis Distributed Locks Documentation](https://redis.io/docs/latest/develop/clients/patterns/distributed-locks/) — official `SET NX EX` pattern + Redlock specification
2. [PostgreSQL `SELECT FOR UPDATE` docs](https://www.postgresql.org/docs/current/explicit-locking.html) — row-level locking semantics
3. [Stormatics: Reduce Contention with `SELECT FOR UPDATE`](https://stormatics.tech/blogs/select-for-update-in-postgresql) — practical tuning
4. [GeeksforGeeks: Design BookMyShow](https://www.geeksforgeeks.org/system-design/design-bookmyshow-a-system-design-interview-question/) — common LLD reference
5. [Medium: BookMyShow LLD by Vrinda Goyal](https://medium.com/@vrindag/bookmyshow-low-level-design-abb4cc995ff2) — class diagram + entity model
6. [Medium: Building a Seat Reservation System (Women in Technology)](https://medium.com/womenintechnology/building-a-seat-reservation-system-deadlock-avoidance-and-transaction-isolation-levels-cad7186eb589) — deadlock avoidance patterns
7. [Martin Kleppmann: How to do distributed locking](https://martin.kleppmann.com/2016/02/08/how-to-do-distributed-locking.html) — the famous Redlock critique (read this if you're being interviewed by a senior engineer)
8. [Coudo AI: BookMyShow real-time ticketing](https://www.coudo.ai/blog/bookmyshow-system-design-how-to-handle-real-time-ticketing) — system design overview

---

**Comment "SEAT" on the reel and I'll DM you this full breakdown doc.**

Follow [@techvijayforyou](https://instagram.com/techvijayforyou) for more.
