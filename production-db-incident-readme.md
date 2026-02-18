# How My Train Journey Almost Broke Our Production Database

## A Real Story From Our Production System

*One small UPDATE query + bad network = 15 minutes of total chaos*

---

## What Happened That Day

**Location:** Somewhere in the train  
**Duration:** 15 minutes of pain  
**Impact:** Database fully loaded, but ride booking was working fine

### The Story

So picture this - I'm traveling in a train, connected via mobile hotspot, and running a simple UPDATE query on our production database:

```sql
BEGIN;
UPDATE ride_requests SET status = 'processed' WHERE created_at < '2024-01-01';
-- Right here, the network vanished! 💀
-- Terminal froze
```

**What Happened:**
1. **Network issue** - Train + mobile data = unstable connection
2. **Hot table** - The `ride_requests` table was under heavy load, getting thousands of requests every second
3. **Lock acquired** - My `BEGIN` started and the UPDATE locked the table
4. **Connection dropped** - Network gone, but PostgreSQL thinks I'm still connected
5. **Queue formed** - Other queries started lining up behind my stuck transaction
6. **Database crashed** - Connection pool full, CPU hit 100%
7. **System down** - Services depending on the database went offline

### But Ride Booking Was Still Working! 🎉

The best part was that **our ride flow was working perfectly fine.**

Why? Because our ride flow runs on a **KV (Key-Value) architecture** that doesn't depend on PostgreSQL. While the database was burning, riders were still booking rides and drivers were still getting requests.

---

## Technical Understanding

### What is PostgreSQL Lock?

When you run `BEGIN`, PostgreSQL starts a transaction. When running an UPDATE:

- **Row lock** - The row being updated gets locked
- **Table lock** - Sometimes the entire table gets locked

**The Problem:** If the connection drops unexpectedly (network failure, laptop sleep, etc.), PostgreSQL doesn't know. It thinks the user is still connected and keeps holding the lock indefinitely.

```
Your Query → Takes Lock → Network Gone
     ↓
PostgreSQL keeps waiting... and waiting... and waiting...
     ↓
Other queries queue up → Pool full → Everything stops
```

### The Lock Chain Reaction

```
0 seconds:   Your UPDATE starts, takes lock
5 seconds:   50 new ride requests arrive → get queued
10 seconds:  200 more requests → queue getting longer
30 seconds:  100 connections full
1 minute:    New connections getting rejected
5 minutes:   Entire system down
15 minutes:  Network comes back, panic! 😰
```

---

## The Fix: Three Settings That Save Lives

As soon as network came back, we immediately applied these settings:

### 1. Statement Timeout

**What it does:** If any query runs longer than 30 seconds, kill it

```sql
-- Global setting (in postgresql.conf)
statement_timeout = '30s'

-- Or per session
SET statement_timeout = '30s';
```

**Why 30 seconds?**
- 99.9% queries finish in under 5 seconds
- Gives buffer for complex queries
- Stops runaway queries

### 2. Lock Timeout

**What it does:** If a query can't acquire a lock in 10 seconds, give an error

```sql
-- Global setting
lock_timeout = '10s'

-- Or per session
SET lock_timeout = '10s';
```

**Why it's needed:**
- Won't wait indefinitely behind stuck transactions
- Fast failure = fast recovery
- Client gets error immediately instead of hanging forever

### 3. Idle in Transaction Timeout

**This is the most important one for our incident!**

**What it does:** If a transaction is open but doing nothing for 60 seconds, close it

```sql
idle_in_transaction_session_timeout = '60s'
```

**Why it's crucial:**
- Catches abandoned transactions (like my network drop situation)
- 60 seconds is enough for interactive use
- No need to manually kill queries

---

## Checklist Before Running Any Query in Production

### First Check These

```sql
-- 1. First apply safety settings
SET statement_timeout = '30s';
SET lock_timeout = '10s';

-- 2. Check how busy the table is
SELECT 
    relname,
    n_tup_ins + n_tup_upd + n_tup_del as total_writes
FROM pg_stat_user_tables 
WHERE relname = 'ride_requests';

-- 3. Check current locks
SELECT 
    locktype,
    relation::regclass,
    mode,
    pid,
    state
FROM pg_locks l
JOIN pg_stat_activity a ON l.pid = a.pid
WHERE relation::regclass::text = 'ride_requests';

-- 4. Check active connections
SELECT 
    count(*) as total_connections,
    count(*) FILTER (WHERE state = 'active') as active,
    count(*) FILTER (WHERE state = 'idle in transaction') as stuck_transactions
FROM pg_stat_activity;
```

### Safe Query Pattern

**Wrong way:**
```sql
BEGIN;
-- Don't update entire table at once!
UPDATE ride_requests SET status = 'completed' WHERE created_at < NOW() - INTERVAL '1 year';
COMMIT;
```

**Right way:**
```sql
-- Update in batches with LIMIT
DO $$
DECLARE
    rows_updated INT;
BEGIN
    LOOP
        UPDATE ride_requests 
        SET status = 'completed' 
        WHERE id IN (
            SELECT id 
            FROM ride_requests 
            WHERE status != 'completed' 
            AND created_at < NOW() - INTERVAL '1 year'
            LIMIT 1000
            FOR UPDATE SKIP LOCKED
        );
        
        GET DIAGNOSTICS rows_updated = ROW_COUNT;
        EXIT WHEN rows_updated = 0;
        
        COMMIT;
        PERFORM pg_sleep(0.1); -- Small pause between batches
    END LOOP;
END $$;
```

---

## Monitoring Queries for Production

Save these - run them regularly or set up alerts:

### 1. Long Running Queries

```sql
SELECT 
    pid,
    now() - query_start as duration,
    state,
    left(query, 100) as query_preview
FROM pg_stat_activity
WHERE state != 'idle'
AND query_start < now() - interval '30 seconds'
ORDER BY query_start;
```

### 2. Lock Wait (This Shows Where the Problem Is)

```sql
SELECT 
    blocked_locks.pid AS blocked_pid,
    blocked_activity.usename AS blocked_user,
    blocking_locks.pid AS blocking_pid,
    blocking_activity.usename AS blocking_user,
    blocked_activity.query AS blocked_query,
    blocking_activity.query AS blocking_query
FROM pg_catalog.pg_locks blocked_locks
JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_catalog.pg_locks blocking_locks ON blocking_locks.pid != blocked_locks.pid
JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted;
```

### 3. Idle in Transaction (Kill These!)

```sql
SELECT 
    pid,
    usename,
    application_name,
    state,
    age(now(), xact_start) as transaction_age,
    left(query, 100) as query_preview
FROM pg_stat_activity
WHERE state = 'idle in transaction'
AND xact_start < now() - interval '1 minute'
ORDER BY xact_start;
```

### 4. Emergency Kill Script

```sql
-- Generate commands to kill stuck queries
SELECT 
    'SELECT pg_terminate_backend(' || pid || '); -- ' || 
    state || ' for ' || age(now(), query_start) || 
    ': ' || left(query, 50) as kill_command
FROM pg_stat_activity
WHERE (state = 'idle in transaction' AND xact_start < now() - interval '5 minutes')
   OR (state = 'active' AND query_start < now() - interval '10 minutes');
```

---

## How KV Architecture Saved Us

### Traditional Architecture (What Failed)

```
Rider App → API Service → PostgreSQL
                              ↓
                         LOCKED! 💀
                              ↓
                    Everything stops
```

### Our Architecture (What Worked)

```
Rider App → API Service → Redis (KV) → Background Sync → PostgreSQL
                              ↓
                    PostgreSQL LOCKED 💀
                              ↓
                    But ride flow working! ✅
```

**Main Lesson:** Keep hot path (ride booking, driver matching) separate from cold path (analytics, reporting).

### Resilience Pattern

| Component | Technology | What It Does | If It Fails |
|-----------|-----------|----------------|----------------|
| Hot Path | Redis/KV | Real-time booking | Nothing (independent) |
| Warm Path | PostgreSQL | Transaction records | Delayed sync, data safe |
| Cold Path | Data Warehouse | Analytics | No immediate impact |

---

## Lessons Learned

### 1. Production Access is a Privilege

- Use read replicas for analytics
- Use connection pooling (PgBouncer)
- Keep VPN + MFA mandatory
- Log all production queries

### 2. Setting Timeouts is a Must

Set these at database level:

```ini
# postgresql.conf
statement_timeout = 30s
lock_timeout = 10s
idle_in_transaction_session_timeout = 60s
```

### 3. Keep Monitoring Always

Set up alerts for:
- Queries running >30 seconds
- Idle in transaction >1 minute
- Lock wait >10 seconds
- Connection pool >80% full

### 4. The 5-Minute Rule

Before running any query in production:
- ⏱️ Will this take >5 minutes? → Run on replica
- 🔥 Is this table busy? → Use `SKIP LOCKED` or off-peak hours
- 🛡️ Are timeouts set? → `SET statement_timeout = '30s'`
- 👀 Is someone watching? → Don't run and disconnect
- 🚪 Do you have an exit plan? → Know how to kill the query

---

## Quick Reference

### Emergency Commands

```bash
# Connect with safety timeouts
psql "postgresql://user:pass@host/db?options=--statement_timeout%3D30s%20--lock_timeout%3D10s"

# In psql session
SET statement_timeout = '30s';
SET lock_timeout = '10s';
```

### Kill Stuck Queries

```sql
-- Find the culprit
SELECT pid, state, query_start, query 
FROM pg_stat_activity 
WHERE state = 'idle in transaction' 
ORDER BY xact_start;

-- Terminate
SELECT pg_cancel_backend(pid);  -- Try this first
SELECT pg_terminate_backend(pid); -- If above doesn't work
```

### Safe Query Template

```sql
-- Always wrap production queries
BEGIN;
SET LOCAL statement_timeout = '30s';
SET LOCAL lock_timeout = '10s';

-- Your query here
UPDATE ... 
WHERE id IN (SELECT id FROM ... LIMIT 1000 FOR UPDATE SKIP LOCKED);

COMMIT;
```

---

## Conclusion

A 15-minute train journey taught us more about database resilience than any training course could. Main takeaways:

1. **Always set timeouts** - They are your safety net
2. **Monitor continuously** - You can't fix what you can't see  
3. **Architect for failure** - Our KV layer saved us when PostgreSQL failed
4. **Respect production** - It's not your personal playground

That day, we lost database connectivity but didn't lose a single ride booking. That's the power of resilient architecture.

**Remember:** It's not about preventing failures - it's about stopping them from cascading.

---

## Resources

- [PostgreSQL Lock Monitoring](https://wiki.postgresql.org/wiki/Lock_Monitoring)
- [PostgreSQL Timeout Configuration](https://www.postgresql.org/docs/current/runtime-config-client.html)

---

*Have you faced similar production incidents? Would love to hear your stories and lessons learned!*

**Author:** [Your Name]  
**Company:** [Your Company]  
**Date:** [Date]

---

#Production #Database #PostgreSQL #Incident #KV #DevOps #Engineering #TrainStory
