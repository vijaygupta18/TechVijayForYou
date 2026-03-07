# UPI: How India Processes 700 Million Transactions a Day Without Crashing

India's Unified Payments Interface handled **21.70 billion transactions in January 2026** — roughly 700 million per day, 7,500 every second at peak. The IMF ranked it the world's largest retail fast-payment system by volume. UPI accounts for **49% of all global real-time digital payment transactions**. And it costs zero to use.

This is the architecture behind it.

---

## The Problem Before UPI

Before 2016, inter-bank transfers in India were painful:

- **NEFT** — batch processing every 30 minutes (now every hour). You'd wait hours. Needed account number + IFSC. One typo = money to a stranger.
- **IMPS** — instant, but required the receiver's 10-digit MMID + mobile number. Nobody memorised that.
- **Net banking** — different UI per bank, session timeouts, no mobile-native experience.

The core problem: **no common identity layer across banks.** Each bank was an island. Inter-bank transfer required you to know the bank's internal identifiers.

UPI solved this with a single abstraction: the **Virtual Payment Address (VPA)** — `vijay@oksbi`, `merchant@ybl`. Human-readable, bank-agnostic, memorable.

---

## The Four-Party Model

UPI operates on a four-party interoperable architecture:

![UPI Four-Party Architecture](images/upi-architecture/upi-four-party.png)

### 1. TPAP (Third Party Application Provider)
PhonePe, Google Pay, CRED, Paytm. These are **apps, not banks.** They never store your UPI PIN, never hold your money, never have banking licenses. They maintain device-to-VPA mappings and transaction UX. Even if a TPAP's servers were fully compromised, attackers would find no bank credentials.

### 2. PSP (Payment Service Provider)
A licensed bank that gives a TPAP access to UPI infrastructure. PhonePe used Yes Bank historically; Google Pay uses Axis Bank and SBI. The PSP is responsible for:
- Registering VPAs with NPCI on behalf of users
- Relaying encrypted transaction messages between TPAP and NPCI
- Regulatory compliance and grievance handling

### 3. NPCI Switch
The central routing and orchestration engine operated by NPCI. This is the brain. It:
- Maintains the **VPA-to-bank-handle mapping registry** (the address book for every UPI ID in India)
- Routes every transaction between the payer's PSP and payee's PSP
- Coordinates the debit and credit instructions in sequence
- Manages settlement cycles

### 4. Issuer Bank / CBS (Core Banking System)
Your actual bank — where your account lives. The CBS exposes just **3 interfaces** to NPCI's UPI adapter:
1. Customer lookup (does this account exist?)
2. Balance check
3. Debit from account / Credit to account

This minimal interface design is what enabled 550+ banks to onboard UPI. Even a small cooperative bank can integrate with UPI if it can implement these 3 operations.

---

## VPA: The Key Abstraction

A VPA looks like: `username@handle`

Examples: `vijay@oksbi`, `merchant@ybl`, `9876543210@paytm`

The **handle** (`@oksbi`, `@ybl`, `@okaxis`) identifies which PSP manages the VPA. NPCI maintains a central **handle-to-PSP mapping registry** — when a transaction arrives for `friend@okhdfc`, NPCI knows immediately that HDFC Bank's UPI server is the destination.

The VPA is mapped to a specific account number + IFSC internally. The payer never needs to know either. One person can have multiple VPAs across multiple PSP apps, linked to the same or different bank accounts.

---

## The Transaction Flow

![UPI Transaction Flow](images/upi-architecture/upi-transaction-flow.png)

**Scenario: PhonePe user (Axis Bank) sends ₹500 to Google Pay user (HDFC Bank)**

### One-time Setup (Device Binding)
1. User installs PhonePe → app generates a **device fingerprint** (device ID + IMEI + SIM data)
2. App sends an SMS from the registered number to NPCI's verification server — confirming SIM + device are the same physical entity
3. User links bank account via debit card → bank calls NPCI's `setCredentials` API
4. User sets a 4 or 6-digit UPI PIN → Axis Bank stores only the **PBKDF2 hash** (600,000 iterations). The PIN itself is never stored anywhere.
5. App generates a **public-private key pair** → private key lives in **Android Keystore / iOS Secure Enclave** (hardware-backed, non-extractable). Public key registered with the PSP.

### Live Transaction (every time you pay)

| Step | What Happens | Who |
|---|---|---|
| 1 | User enters payee VPA + amount, confirms with UPI PIN | User on PhonePe |
| 2 | PhonePe app encrypts PIN, signs request with device private key | PhonePe (TPAP) |
| 3 | Encrypted request → PhonePe's PSP (Axis Bank's UPI server) | PhonePe → PSP |
| 4 | PSP forwards to NPCI Switch via ISO 8583 XML over HTTPS | PSP → NPCI |
| 5 | NPCI resolves `friend@okhdfc` → HDFC Bank is payee PSP | NPCI |
| 6 | NPCI sends VPA resolution request to HDFC; HDFC confirms account exists | NPCI → HDFC |
| 7 | NPCI instructs Axis Bank: "Verify PIN and debit ₹500 from account XXXX" | NPCI → Axis |
| 8 | Axis Bank CBS verifies PIN hash against stored hash, checks balance, debits account → sends SUCCESS | Axis Bank |
| 9 | NPCI instructs HDFC Bank: "Credit ₹500 to account YYYY" | NPCI → HDFC |
| 10 | HDFC Bank CBS credits account → sends SUCCESS | HDFC Bank |
| 11 | NPCI sends confirmation back to both PSPs | NPCI |
| 12 | PhonePe notifies payer; Google Pay notifies payee | TPAPs |

**End-to-end target latency: under 2 seconds.** NPCI reduced the API response time cap from 30s → 15s in June 2025, with further reductions planned.

---

## Two-Factor Authentication: Why UPI Is Hard to Compromise

UPI requires **both** factors simultaneously:

### Factor 1 — Possession (Device Binding)
- Your registered SIM + specific device = bound pair
- The app's private key lives in **ARM TrustZone** (Android) or **Secure Enclave** (iOS) — hardware-level isolation, the key cannot be extracted even by the OS
- Every transaction request is cryptographically signed by this private key
- NPCI verifies the signature against the public key on file
- New phone = full re-registration required

### Factor 2 — Knowledge (UPI PIN)
- PIN is entered in a **bank-provided SDK overlay** — the TPAP app never sees it
- Bank stores only the PBKDF2 hash (600,000 iterations) — not the PIN, not an encrypted PIN
- PIN verification happens inside the bank's **HSM (Hardware Security Module)** — not on NPCI's servers

**What this means:** Stealing just your phone doesn't work (you need the PIN). Stealing just the PIN doesn't work (you need the bound device). A server-side compromise of the TPAP reveals nothing — they have no credentials. A server-side compromise of NPCI reveals nothing — PIN hashes never reach NPCI.

---

## The Settlement Trick: What "Success" Actually Means

When PhonePe shows "Payment Successful," your Axis Bank account has been debited and your friend's HDFC account has been credited in their respective CBS systems — **in real time.** This is visible immediately.

But **Axis Bank and HDFC Bank have not exchanged actual funds yet.**

Here's what happens in the background:

![UPI Settlement Cycles](images/upi-architecture/upi-settlement.png)

NPCI runs **10 daily settlement cycles** (per NPCI OC-197, implemented FY2024-25), from 9 AM to 9 PM at approximately 2-hour intervals:

1. NPCI aggregates all transactions across all bank pairs in that window
2. Calculates net positions: "Axis owes HDFC ₹847 crore net, HDFC owes ICICI ₹312 crore net..."
3. Net settlement instructions posted to **RBI's RTGS** (Real-Time Gross Settlement)
4. Banks transfer only the net amount via their RBI reserve accounts

**The math:** If in a 2-hour window, Axis→HDFC transactions total ₹100 crore and HDFC→Axis total ₹95 crore, only **₹5 crore moves via RTGS** — not 200 crore worth of individual transactions. This dramatically reduces inter-bank liquidity requirements and makes the system capital-efficient.

Banks carry the intraday credit risk (crediting payees before actual inter-bank settlement) based on NPCI's guarantee, which is backed by RBI's oversight and the DNS settlement rules.

In November/December 2025, NPCI added 2 separate **dispute settlement cycles** (DC1, DC2) — segregated from the 10 auth cycles, so failed transaction reversals don't contaminate normal settlement windows.

---

## UPI vs Everything Else

![Global Payments Share](images/upi-architecture/upi-global-share.png)

| Dimension | UPI | IMPS | NEFT | SWIFT |
|---|---|---|---|---|
| User experience | Instant (< 2s) | Instant (seconds) | Near-instant (batch) | Hours to days |
| Identifier | VPA (human-readable) | Account + IFSC | Account + IFSC | IBAN + SWIFT BIC |
| Auth | 2FA (device + PIN) | Password + OTP | Net banking | Correspondent trust |
| Settlement | Deferred Net (10 cycles/day) | Deferred Net | Batch | Correspondent chain |
| Cost to user | Zero | ₹1–25 | ₹1–25 | High (wire + FX) |
| Daily limit | ₹1L (₹5L select categories) | ₹5L | No limit | No standard |
| Availability | 24/7/365 | 24/7/365 | 24/7 (since Dec 2019) | Banking hours |

**Why UPI is fundamentally different from SWIFT:** SWIFT is a messaging network — it sends payment instructions but doesn't move money. Each bank along the correspondent chain holds funds and messages the next. UPI is a **centralized switch model** — NPCI orchestrates debit and credit in a single coordinated sequence, no correspondent chain, single hop.

---

## Infrastructure at Scale

**Data Centers (NPCI official press releases, 2020):**
- Two Tier IV data centers: Chennai (SIPCOT IT Park, Siruseri) and Hyderabad (Narsingi Village)
- Tier IV = 99.995% uptime at facility level (2-failure tolerant design)
- Active-active setup — traffic fails over automatically if one DC goes down
- Combined capacity: 50-70 billion transactions per month

**Technology stack (Kyndryl partnership, publicly announced):**
- Internal infrastructure: Terraform + Ansible + GitHub Runners + MAAS (Canonical) — NPCI calls it the "TARM stack"
- Protocol: ISO 8583 XML over HTTPS — the same format used in card payment networks, chosen because it accommodates limited bandwidth from small banks' CBS systems

**Scale controls (NPCI August 2025 circular):**
- NPCI introduced daily limits on balance enquiry and "list accounts" API calls
- PSPs required to rate-limit transaction initiation TPS to prevent spike overloads

**Published uptime:** 99.2% transaction success rate (includes all failure reasons — bank timeouts, CBS rejections, network drops). NPCI publishes month-wise uptime data at npci.org.in.

---

## Fraud Prevention

**MuleHunter.AI** — NPCI provides free ML tools to partner banks that analyze transaction patterns, device details, and behavioral anomalies in real time.

**Financial Fraud Risk Indicator (FRI)** — DoT launched this in May 2025. Classifies mobile numbers as Medium/High/Very High fraud risk based on cybercrime reports, telecom intelligence, and bank data. Available at transaction time.

**What UPI's architecture prevents by design:**
- Replay attacks: every transaction has a unique ID + timestamp
- Man-in-the-middle: end-to-end encryption + signed intents (QR codes carry merchant cryptographic signatures)
- Phishing: PIN entered in bank-controlled SDK, not TPAP UI
- SIM swap: device binding requires full re-registration from new SIM, triggering bank verification

---

## The Global Picture

India's UPI in 2025:
- 228.3 billion transactions, worth ₹299.7 lakh crore (~$3.6 trillion) — 29.3% YoY growth
- 49% of all global real-time digital payment transactions (ACI Worldwide, IMF 2025)
- India alone: 228.3B vs Brazil (37.4B) + Thailand (20.4B) + China (17.2B) + South Korea (9.1B) combined

NPCI's cross-border expansion: 40+ countries onboarding UPI acceptance. NPCI + NVIDIA partnership (Feb 2026) building an AI layer on top of UPI for fraud detection and credit scoring.

---

## Why UPI's Architecture Is Worth Studying

Three design decisions made UPI work at this scale:

1. **Centralized switch, decentralized accounts** — NPCI owns routing, banks own accounts. Clear separation of concerns. No bank has to trust another bank directly.

2. **Minimal CBS integration** — 3 operations (lookup, balance, debit). Even a 30-year-old bank CBS can implement this. Network effects scaled because the bar to join was low.

3. **Deferred Net Settlement** — instant experience for users, batch settlement for banks. Avoids requiring banks to maintain massive intraday liquidity buffers. Settlement risk managed by NPCI's guarantee, backed by RBI.

The elegance is that none of these individually is novel — what NPCI built was the right combination of trade-offs for the Indian banking system's specific constraints.

---

## References

- [NPCI UPI Product Statistics](https://www.npci.org.in/product/upi/product-statistics)
- [NPCI OC-197: 10 Settlement Cycles](https://www.npci.org.in/PDF/npci/upi/circular/2024/UPI-OC-No-197-FY-24-25-Implementation-of-10-settlement-cycles-&-revised-business-day-cutover.pdf)
- [NPCI OC-222A: Dispute Settlement Cycles](https://www.npci.org.in/uploads/UPI_OC_No_222_A_FY_2025_26_Addendum_segregation_of_UPI_settlement_cycles_for_Auth_and_disputes_transactions_42766ab9aa.pdf)
- [PIB: IMF Recognises UPI as World's Largest Real-Time Payment System](https://www.pib.gov.in/PressReleasePage.aspx?PRID=2200569&reg=3&lang=1)
- [PIB: DoT Introduces Financial Fraud Risk Indicator](https://www.pib.gov.in/PressReleasePage.aspx?PRID=2130249)
- [NPCI UPI Settlement Process (PDF)](https://www.npci.org.in/PDF/npci/others/UPI-Settlement-Process.pdf)
- [NPCI Chennai Data Center Press Release](https://www.npci.org.in/PDF/npci/press-releases/2020/NPCI%20Press%20Release%20-%20NPCI%20to%20launch%20Smart%20Data%20Center%20in%20Chennai.pdf)
- [NPCI Hyderabad Data Center Press Release](https://www.npci.org.in/PDF/npci/press-releases/2020/NPCI_Press_Release-NPCI_to_launch_Smart_Data_Center_in_Hyderabad.pdf)
- [NPCI + Kyndryl Data Centre Modernisation](https://www.itnews.asia/news/indias-npci-modernises-data-centres-using-kyndryls-cloud-services-592061)
- [ACI Worldwide: Prime Time for Real-Time 2024](https://www.aciworldwide.com/prime-time-for-real-time)
- [Setu: UPI 102 — The Transaction Cycle](https://blog.setu.co/articles/upi-102-the-transaction-cycle)
- [UPI Smashes Records — 21.6B Transactions December 2025](https://bfsi.eletsonline.com/upi-smashes-records-with-21-6-billion-transactions-in-december-2025/)
- [World Bank: India IMPS and UPI Case Study](https://fastpayments.worldbank.org/sites/default/files/2021-10/World_Bank_FPS_India_IMPS_and_UPI_Case_Study.pdf)

---

#systemdesign #upi #softwareengineer #india #fintech #distributedsystems #npci #payments #coding #techincident
