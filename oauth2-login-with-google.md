# OAuth2 Flow — How "Login with Google" Actually Works

You press "Login with Google" every day. One click, you're in. Simple, right?

Behind that button, **6 invisible network calls** happen before your app knows who you are. Most developers implement OAuth2 without understanding any of them. And in interviews? OAuth2 is asked at every company that builds auth — Amazon, Flipkart, Razorpay, PhonePe, Stripe.

Let's trace the complete journey — from that button click to you being logged in.

---

## The Wrong Way First — Why Not Just Send the Password?

The naive approach: user types their Google password into YOUR app. Your app sends it to Google. Google says "yep, that's them."

Why this is terrible:
- **Your app sees the password.** If you get hacked, every user's Google password leaks.
- **No scoped access.** Your app gets full access to the user's Google account — emails, photos, everything.
- **No revocation.** User can't revoke access to just your app without changing their Google password.

This is exactly the problem OAuth2 was designed to solve. Published as [RFC 6749](https://datatracker.ietf.org/doc/html/rfc6749) in October 2012, OAuth2 is now used by virtually every major web service — Google, GitHub, Facebook, Microsoft, Apple, Spotify, Slack, and more.

> **The core idea:** Your app never sees the user's password. Instead, Google gives your app a **limited-time, limited-scope token** — like a hotel keycard instead of the master key.

---

## The 4 Players (RFC 6749, Section 1.1)

Before diving into the flow, meet the 4 roles:

| Role | Who | Example |
|------|-----|---------|
| **Resource Owner** | The end user | You, clicking "Login with Google" |
| **Client** | The application requesting access | Your web/mobile app |
| **Authorization Server** | Issues tokens after authentication | `accounts.google.com` |
| **Resource Server** | Hosts protected resources (APIs) | `googleapis.com/userinfo` |

Google plays both the Authorization Server and the Resource Server. Your app is the Client. You are the Resource Owner.

---

## The Complete Flow — 6 Steps, Traced

This is the **Authorization Code flow with PKCE** — the gold standard in 2026. Let's trace every step.

![OAuth2 Authorization Code Flow](images/oauth2-login-with-google/auth-code-flow.png)

<p align="center">
  <img src="images/oauth2-login-with-google/oauth2-flow-animated.svg" alt="Animated OAuth2 Authorization Code flow — a glowing dot travels from User to App to Google and back, showing the 6-step token exchange with PKCE code_challenge, authorization code, and three tokens returned" width="700" />
</p>

Watch the glowing dot trace the complete journey: User clicks login → App redirects to Google with PKCE code_challenge → Google returns authorization code → App exchanges code server-to-server → Three tokens returned. The entire flow loops every 8 seconds.

### Step 1 — User Clicks "Login with Google"

Your app generates two cryptographic values (this is the **PKCE** part — [RFC 7636](https://datatracker.ietf.org/doc/html/rfc7636), published September 2015):

```
code_verifier = random_string(43-128 characters)
code_challenge = BASE64URL(SHA256(code_verifier))
```

The `code_verifier` is a secret your app keeps. The `code_challenge` is a one-way hash of it — safe to send over the network because SHA-256 can't be reversed.

Then your app redirects the user's browser to Google:

```
GET https://accounts.google.com/o/oauth2/v2/auth
  ?response_type=code
  &client_id=your_app_id
  &redirect_uri=https://yourapp.com/callback
  &scope=openid profile email
  &state=random_csrf_token
  &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM
  &code_challenge_method=S256
```

Key parameters:
- **`response_type=code`** — "give me an authorization code, not a token directly"
- **`scope=openid profile email`** — "I only want to know who this user is — name and email. Not their Google Drive files."
- **`state`** — a random value for CSRF protection. Your app will verify this matches when Google redirects back.
- **`code_challenge`** — the SHA-256 hash of `code_verifier`. Google stores this for later verification.

### Step 2 — User Authenticates with Google

Google shows its login page. The user enters their Google email and password **directly on Google's page** — your app never sees these credentials. Google may also show a consent screen: "This app wants to see your name and email. Allow?"

The user clicks "Allow."

### Step 3 — Google Redirects Back with an Authorization Code

Google redirects the user's browser to your `redirect_uri`:

```
HTTP/1.1 302 Found
Location: https://yourapp.com/callback
  ?code=SplxlOBeZQQYbYS6WxSbIA
  &state=random_csrf_token
```

The `code` is a **short-lived, single-use** authorization code. Per RFC 6749 §4.1.2:
- Maximum recommended lifetime: **10 minutes**
- Must be **single-use** — if the same code is presented twice, the server must reject it and should revoke all tokens issued from that code

Your app first checks that the `state` matches what it sent in Step 1 (CSRF protection). If it doesn't match — abort. Someone might be trying to inject a foreign authorization code.

> **Why a code and not a token directly?** Because this redirect goes through the user's browser — it's visible in the URL bar, browser history, and server logs. The code alone is useless without the `code_verifier` (PKCE) or `client_secret`. The actual token exchange happens server-to-server in the next step.

### Step 4 — Your App Exchanges the Code for Tokens

Your app makes a **server-to-server POST** (not through the browser) to Google's token endpoint:

```
POST https://oauth2.googleapis.com/token
Content-Type: application/x-www-form-urlencoded

grant_type=authorization_code
&code=SplxlOBeZQQYbYS6WxSbIA
&redirect_uri=https://yourapp.com/callback
&client_id=your_app_id
&client_secret=your_app_secret
&code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
```

Google verifies:
1. The `code` is valid and hasn't been used before
2. The `redirect_uri` matches exactly what was sent in Step 1
3. The `client_secret` matches the registered app
4. **PKCE check:** `BASE64URL(SHA256(code_verifier)) == code_challenge` from Step 1

If everything checks out, Google responds with tokens:

```json
{
  "access_token": "ya29.a0AfH6SMBx...",
  "token_type": "Bearer",
  "expires_in": 3920,
  "refresh_token": "1//0gdBjS2e...",
  "scope": "openid profile email",
  "id_token": "eyJhbGciOiJSUzI1NiIsI..."
}
```

Google issues access tokens with `expires_in: 3920` — roughly **65 minutes**.

### Step 5 — Your App Uses the Access Token

Your app can now call Google's APIs with the access token:

```
GET https://www.googleapis.com/oauth2/v2/userinfo
Authorization: Bearer ya29.a0AfH6SMBx...
```

Response:
```json
{
  "id": "1234567890",
  "email": "user@gmail.com",
  "name": "Vijay Gupta",
  "picture": "https://lh3.googleusercontent.com/..."
}
```

The `Authorization: Bearer <token>` header is defined in [RFC 6750](https://datatracker.ietf.org/doc/html/rfc6750). "Any party in possession of a bearer token can use it to get access to the associated resources" — which is exactly why access tokens are short-lived.

### Step 6 — Refresh Token Rotation

After ~65 minutes, the access token expires. Instead of making the user log in again, your app uses the refresh token:

```
POST https://oauth2.googleapis.com/token
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token
&refresh_token=1//0gdBjS2e...
&client_id=your_app_id
&client_secret=your_app_secret
```

Google returns a new access token (and possibly a new refresh token). This cycle continues until the user revokes access.

**Refresh Token Rotation** (RFC 6749 §10.4): Each time a refresh token is used, a new one is issued and the old one is invalidated. Why? If an attacker steals a refresh token and uses it, one of two things happens — either the attacker or the legitimate app will present the old (already-rotated) token. The server detects this and can invalidate the entire token family. It's a built-in theft detection mechanism.

---

## What's Inside a JWT? (RFC 7519)

The `id_token` in Step 4's response is a **JWT** (JSON Web Token, pronounced "jot") — defined in [RFC 7519](https://datatracker.ietf.org/doc/html/rfc7519), published May 2015.

A JWT has 3 parts separated by dots:

```
eyJhbGciOiJSUzI1NiJ9.eyJpc3MiOiJhY2NvdW50cy5nb29nbGUuY29tIiwiZW1haWwiOiJ1c2VyQGdtYWlsLmNvbSJ9.signature_here
```

```
BASE64URL(Header) . BASE64URL(Payload) . BASE64URL(Signature)
```

![JWT Structure](images/oauth2-login-with-google/jwt-structure.png)

### Part 1 — Header
```json
{
  "alg": "RS256",
  "typ": "JWT"
}
```
Tells you the signing algorithm. RS256 = RSA + SHA-256.

### Part 2 — Payload (Claims)
```json
{
  "iss": "accounts.google.com",
  "sub": "1234567890",
  "aud": "your_app_id",
  "exp": 1700000000,
  "iat": 1699996400,
  "email": "user@gmail.com",
  "name": "Vijay Gupta"
}
```

The 7 registered claims (RFC 7519 §4.1):

| Claim | Full Name | Purpose |
|-------|-----------|---------|
| `iss` | Issuer | Who issued this token (`accounts.google.com`) |
| `sub` | Subject | Who this token is about (user ID) |
| `aud` | Audience | Who this token is for (your app's client_id) |
| `exp` | Expiration | When the token expires (Unix timestamp) |
| `nbf` | Not Before | Token is invalid before this time |
| `iat` | Issued At | When the token was created |
| `jti` | JWT ID | Unique token identifier (replay prevention) |

### Part 3 — Signature

```
RSA-SHA256(
  BASE64URL(header) + "." + BASE64URL(payload),
  google_private_key
)
```

Your app verifies the signature using Google's public key (available at `https://www.googleapis.com/oauth2/v3/certs`). If the signature is valid, the token hasn't been tampered with.

> **CRITICAL: Base64 is NOT encryption.** Anyone can decode the header and payload — just paste a JWT into [jwt.io](https://jwt.io) and see. The signature only guarantees **integrity** (hasn't been modified) and **authenticity** (was issued by Google). It does NOT provide **confidentiality**. Never put secrets in a JWT payload.

---

## Why PKCE Exists — The Authorization Code Interception Attack

PKCE ([RFC 7636](https://datatracker.ietf.org/doc/html/rfc7636)) was created to solve a specific attack on mobile apps.

**The attack:** On mobile, when Google redirects back to your app with the authorization code (Step 3), it uses a custom URI scheme like `myapp://callback?code=XYZ`. A malicious app can register itself as a handler for the same `myapp://` scheme and intercept the code. Since mobile apps can't securely store a `client_secret` (it's embedded in the app binary and can be extracted), the attacker can exchange the stolen code for tokens.

<p align="center">
  <img src="images/oauth2-login-with-google/pkce-attack-prevention.svg" alt="Animated side-by-side comparison — without PKCE the attacker intercepts the authorization code and steals the token; with PKCE the attacker is rejected because they cannot reverse the SHA-256 code_challenge" width="700" />
</p>

Left side: without PKCE, the attacker intercepts the authorization code and exchanges it for a token — nothing stops them. Right side: with PKCE, the attacker has the code but not the code_verifier, so Google rejects the exchange. The pulsing red/green borders highlight the attack vs. protection in real time.

**How PKCE prevents it:**
1. Your app generates a random `code_verifier` and sends `SHA256(code_verifier)` as `code_challenge` in Step 1
2. The attacker intercepts the code in Step 3
3. The attacker tries to exchange the code in Step 4 — but doesn't know the original `code_verifier`
4. Google checks: `SHA256(attacker_guess) != stored_code_challenge` → **rejected**

The SHA-256 hash is one-way. Even if the attacker saw the `code_challenge` in Step 1's URL, they can't reverse it to get `code_verifier`.

PKCE is now recommended for ALL client types — not just mobile. Even web apps with a `client_secret` should use PKCE because it also prevents authorization code injection attacks.

---

## Why the Implicit Flow Was Killed

Before Authorization Code + PKCE became the standard, SPAs (Single Page Applications) used the **Implicit flow** (RFC 6749 §1.3.2). In this flow, Google returned the access token directly in the URL fragment:

```
https://yourapp.com/callback#access_token=ya29.a0AfH6SMBx...
```

Five security problems killed it:

| Problem | What Happens |
|---------|-------------|
| **Browser history** | The access token sits in the URL → stored in browser history → anyone with device access can steal it |
| **Referer header leak** | If your callback page loads any third-party resource (analytics, ads, CDN), the URL with the token leaks in the `Referer` header |
| **No client authentication** | Anyone who knows the `client_id` can initiate the flow — no `client_secret` exchange |
| **Token injection** | Attacker substitutes a stolen token in the response — no way to detect the swap |
| **No refresh tokens** | Can't refresh silently — user must re-authenticate every time the token expires |

The OAuth 2.0 Security Best Current Practice (draft-ietf-oauth-security-topics-29, June 2024) now **recommends against using the Implicit flow entirely**. Some OAuth servers have disabled it completely.

---

## Access Token vs Refresh Token — The Lifecycle

| Property | Access Token | Refresh Token |
|----------|-------------|---------------|
| **Purpose** | Access protected APIs | Get new access tokens |
| **Lifetime** | Short: 15 min – 1 hour (Google: ~65 min) | Long: 7 – 90 days (can be perpetual until revoked) |
| **Sent to** | Resource Server (API) | Authorization Server only |
| **Stored where** | Memory (frontend) or server-side session | Server-side only — never in browser |
| **If stolen** | Damage limited to token's short lifetime | Attacker can generate new access tokens indefinitely |
| **Rotation** | New one issued on every refresh | Old one invalidated on each use (theft detection) |

<p align="center">
  <img src="images/oauth2-login-with-google/refresh-token-rotation.svg" alt="Animated refresh token rotation — normal cycle shows expired access token being refreshed with new token pair, then theft detection shows attacker using old rotated token triggering THEFT DETECTED alert and full revocation" width="700" />
</p>

Top half: the normal rotation cycle — expired access token → use refresh token → new token pair issued, old invalidated. Bottom half: when an attacker tries the old (already-rotated) refresh token, the server detects the breach and revokes the entire token family. The pulsing red alert makes the theft detection mechanism visceral.

The design philosophy: access tokens are **disposable**. They're meant to be short-lived and used frequently. Refresh tokens are the **long-term credential** — they're stored securely server-side and rotated on each use to detect theft.

---

## Real-World OAuth2 Breaches

OAuth2 is only as secure as its implementation. Here are real incidents:

### Grammarly, Vidio (100M MAU), Bukalapak — Token Audience Bypass (2023)

Salt Security researchers found that these sites did not verify whether an incoming OAuth access token was actually issued for their specific application. An attacker could take a token from a different app using the same OAuth provider (e.g., Facebook) and use it to log into these sites as any user.

**Root cause:** One missing check — verifying the token's `aud` (audience) claim matches your app's ID. Facebook's docs explicitly say to call `graph.facebook.com/debug_token` for this — these apps skipped it.

**Impact:** Full account takeover across hundreds of millions of users.

**Lesson:** Always verify the `aud` claim. A valid token from the wrong app is just as dangerous as a forged token.

### The `alg: none` Attack (Early 2010s)

Some JWT libraries accepted `"alg": "none"` in the header — meaning the token had no signature. An attacker could forge any JWT by setting `alg` to `none` and removing the signature entirely. The server would accept it as valid.

**Fix:** Always specify allowed algorithms on the server side. Never accept `alg: none`.

---

## The Complete Picture

```
User clicks "Login with Google"
    │
    ▼
[1] App generates code_verifier → SHA256 → code_challenge
    App redirects browser to Google with code_challenge
    │
    ▼
[2] User authenticates on Google's page (app never sees password)
    User grants consent
    │
    ▼
[3] Google redirects back to app with authorization code
    (code alone is useless without code_verifier)
    │
    ▼
[4] App sends code + code_verifier + client_secret to Google (server-to-server)
    Google verifies: SHA256(code_verifier) == code_challenge ✓
    Google returns: access_token + refresh_token + id_token (JWT)
    │
    ▼
[5] App calls Google APIs with access_token
    Gets user profile (name, email, photo)
    │
    ▼
[6] Access token expires (~65 min) → App uses refresh_token → new access_token
    Old refresh_token invalidated → theft detection
```

---

## Interview Quick-Fire — Top 10 OAuth2 Questions

1. **What's the difference between OAuth2 and OpenID Connect?**
   OAuth2 = authorization (delegated access). OIDC = authentication (identity) built on top of OAuth2. OIDC adds the `id_token` (JWT) and `/userinfo` endpoint.

2. **Why Authorization Code over Implicit?**
   Implicit exposes tokens in URL → browser history, referer leaks, no refresh. Auth Code exchanges code server-to-server → token never in browser.

3. **What is PKCE?**
   Proof Key for Code Exchange. Prevents authorization code interception. Client sends `SHA256(code_verifier)` upfront, proves knowledge of `code_verifier` at token exchange.

4. **What's inside a JWT?**
   Header (algorithm) + Payload (claims: iss, sub, aud, exp, iat, nbf, jti) + Signature. Base64-encoded, NOT encrypted.

5. **Access token vs refresh token?**
   Access: short-lived (~1hr), sent to API, disposable. Refresh: long-lived (days-months), sent only to auth server, rotated for theft detection.

6. **What is the `state` parameter?**
   CSRF protection. Random value sent in auth request, verified on callback. Prevents attacker from injecting their authorization code into your session.

7. **Confidential vs public clients?**
   Confidential: server-side apps, can store `client_secret` securely. Public: mobile/SPA, can't hide `client_secret` → must use PKCE.

8. **What is refresh token rotation?**
   New refresh token issued on each use, old one invalidated. If an old rotated token appears → theft detected → entire token family revoked.

9. **Why is `aud` verification critical?**
   Without it, a valid token from App A can be used to log into App B. The Salt Security breaches (Grammarly, Vidio) happened because of this missing check.

10. **What is the `scope` parameter?**
    Principle of least privilege. `openid profile email` = only identity. `repo` (GitHub) = repository access. Request only what you need.

---

## Summary

| Concept | Key Takeaway |
|---------|-------------|
| OAuth2 | App gets limited-time token, never sees user's password |
| Authorization Code + PKCE | Code exchanged server-to-server, PKCE prevents interception |
| JWT | Header.Payload.Signature — base64 encoded, NOT encrypted |
| Access Token | Short-lived (~1hr), disposable, sent to APIs |
| Refresh Token | Long-lived, rotated on each use, detects theft |
| Implicit Flow | Killed — token in URL = browser history/referer leaks |
| `aud` Verification | MUST check — missing check = account takeover |

---

## References

- [RFC 6749 — The OAuth 2.0 Authorization Framework](https://datatracker.ietf.org/doc/html/rfc6749) (October 2012)
- [RFC 7636 — Proof Key for Code Exchange (PKCE)](https://datatracker.ietf.org/doc/html/rfc7636) (September 2015)
- [RFC 7519 — JSON Web Token (JWT)](https://datatracker.ietf.org/doc/html/rfc7519) (May 2015)
- [RFC 6750 — Bearer Token Usage](https://datatracker.ietf.org/doc/html/rfc6750) (October 2012)
- [Google Identity — OAuth2 for Web Server Applications](https://developers.google.com/identity/protocols/oauth2/web-server)
- [Auth0 — Authorization Code Flow with PKCE](https://auth0.com/docs/get-started/authentication-and-authorization-flow/authorization-code-flow-with-pkce)
- [OAuth Security BCP — draft-ietf-oauth-security-topics-29](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-security-topics-29) (June 2024)
- [Salt Security — OAuth Implementation Flaws](https://salt.security/blog/oh-auth-abusing-oauth-to-take-over-millions-of-accounts) (October 2023)
- [Cloudflare — What Happens in a TLS Handshake](https://www.cloudflare.com/learning/ssl/what-happens-in-a-tls-handshake/)

---

#systemdesign #oauth2 #authentication #jwt #softwareengineer #coding #interviewprep #webdevelopment #security #techvijayforyou
