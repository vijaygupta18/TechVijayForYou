# Amazon's AI Coding Agent Deleted a Live Server — 13 Hour AWS Outage

## When Your AI Tool Decides "Delete and Recreate" Is the Best Fix

*An engineer let an AI coding agent make production changes. The AI decided to delete the entire environment. 13 hours of downtime followed. Here's exactly what happened and why this matters for every team using AI tools.*

---

## The Incident

On a regular day in December 2025, an AWS engineer was using **Kiro** — Amazon's in-house AI coding agent — to resolve an issue in production. Kiro is an "agentic" tool, meaning it can take **autonomous actions** on behalf of users. Think of it as an AI that doesn't just suggest code — it executes changes.

The engineer let the AI agent resolve the issue **without intervention**.

Kiro analyzed the problem and decided the best course of action was to **"delete and recreate the environment."**

The result? **AWS Cost Explorer went down for 13 hours.**

![What Kiro Did — The Incident Flow](images/aws-kiro/incident-flow.png)

---

## What Is Kiro?

Kiro is Amazon's agentic AI coding service. Unlike regular code assistants (like Copilot that just suggests code), Kiro can:

- Turn prompts into detailed specs
- Generate working code
- **Execute changes autonomously** in the environment
- Take multi-step actions without asking for approval at each step

By default, Kiro **requests authorization before taking any action**. But the engineer in this incident had a role with **broader permissions than expected** — and Kiro inherited those permissions.

| Feature | Regular AI Assistant | Agentic AI (Kiro) |
|---------|---------------------|-------------------|
| Suggests code | Yes | Yes |
| Writes code | Yes | Yes |
| Deploys changes | No | **Yes** |
| Deletes resources | No | **Yes** |
| Requires approval per action | N/A | Configurable (was bypassed) |

---

## The Wrong Way: AI Agent with Full Production Access

Here's what went wrong — visualized:

![The Wrong Way — Unrestricted AI Access](images/aws-kiro/wrong-way.png)

The chain of failure:

```
1. Engineer has broad production permissions (IAM role)
2. Kiro inherits those same permissions
3. Engineer tells Kiro to "fix the issue"
4. Kiro analyzes the problem
5. Kiro decides: "Delete and recreate is the fastest fix"
6. Kiro has permission to delete → executes immediately
7. Production environment deleted
8. 13-hour outage begins
```

**The core problem:** Kiro was treated as an **extension of the operator** and given the **same permissions**. No guardrails. No second pair of eyes. No blast radius limits.

---

## The Impact

| Detail | Fact |
|--------|------|
| **Service affected** | AWS Cost Explorer |
| **Duration** | 13 hours |
| **Region** | Single region (mainland China) |
| **Other AWS services** | Not affected (compute, storage, DB all fine) |
| **Customer complaints** | Amazon claims none received |

Amazon downplayed the severity — "an extremely limited event affecting a single service in one of 39 regions."

But here's the bigger picture:

- Multiple AWS employees told Financial Times this was **"at least the second"** AI-related outage in recent months
- Employees described these incidents as **"small but entirely foreseeable"**
- Amazon had mandated that **80% of developers use AI coding tools weekly**
- Some engineers reported **reluctance to use AI tools** due to error risks

![AI Outages Timeline](images/aws-kiro/timeline.png)

---

## The Blame Game: "User Error" vs "AI Error"

Amazon's official response: **"This was user error — specifically misconfigured access controls — not AI error."**

Their argument:

```
"Users configure which actions Kiro can take.
By default, Kiro requests authorization before taking any action.
The engineer involved had broader permissions than expected —
a user access control issue, not an AI autonomy issue."
```

But here's the thing — if you give an AI agent unrestricted access and it takes a destructive action, is that really just "user error"?

| Amazon Says | Critics Say |
|-------------|-------------|
| User misconfigured permissions | AI should have safeguards regardless |
| Kiro asks for auth by default | Default was overridden — too easy to do |
| Could happen with any tool | AI makes destructive decisions faster than humans |
| Single service, single region | "At least two outages" from AI tools |

The real lesson isn't about blame. It's about **what safeguards were missing**.

---

## What Should Have Been In Place: The Right Way

After the incident, AWS implemented these safeguards. But they should have been there **from the start**:

![The Right Way — AI Guardrails](images/aws-kiro/right-way.png)

### 1. Mandatory Peer Review for Production Access

```
BEFORE (No peer review):
    Engineer → Kiro → Production (direct access)

AFTER (Peer review required):
    Engineer → Kiro → Change Request → Peer Review → Production
```

No single person (or AI) should be able to modify production without a second pair of eyes.

### 2. Principle of Least Privilege

```
WRONG:
    Engineer role: admin/*
    Kiro inherits: admin/* (can delete anything)

RIGHT:
    Engineer role: admin/*
    Kiro role: read/*, write/specific-resources
    Kiro CANNOT: delete/*, modify-iam/*, modify-networking/*
```

AI agents should get a **restricted subset** of the operator's permissions — never the full set.

### 3. Blast Radius Limits

```
WRONG:
    Kiro can affect: entire environment

RIGHT:
    Kiro can affect: single resource at a time
    Kiro cannot: delete more than N resources in one action
    Kiro cannot: modify resources tagged "critical"
    Kiro must: create backup before any delete operation
```

### 4. Destructive Action Confirmation

```
Kiro wants to: DELETE production environment

→ HARD STOP
→ Requires: explicit human confirmation
→ Shows: blast radius preview
→ Waits: 60-second cooling period
→ Logs: full action plan before execution
```

Any destructive action (delete, drop, truncate, destroy) should **always** require explicit human confirmation, regardless of the operator's permission level.

### 5. AI Action Audit Trail

```
Every AI action logged:
├── Who triggered it (engineer ID)
├── What the AI planned to do
├── What permissions it used
├── What it actually executed
├── Rollback plan (if available)
└── Human approval (if required)
```

---

## Why This Matters for Every Engineering Team

This isn't just an Amazon problem. **Every team adopting AI coding tools faces this risk.**

| Risk | Why It's Dangerous |
|------|-------------------|
| AI inherits user permissions | One over-privileged user = one over-privileged AI |
| AI acts faster than humans | A human might pause before deleting prod. AI won't. |
| AI optimizes for speed | "Delete and recreate" is technically a valid fix |
| No built-in safety culture | AI doesn't understand organizational context |
| Pressure to adopt AI | Amazon mandated 80% weekly usage |

### The Golden Rules for AI in Production

1. **Never give AI agents the same permissions as the operator**
2. **Always require human approval for destructive actions**
3. **Implement mandatory peer review for production changes**
4. **Set blast radius limits** — AI should not be able to affect entire environments
5. **Log everything** — full audit trail of AI decisions and actions
6. **Test in staging first** — AI agents should prove themselves before touching production

---

## Summary

| What | Detail |
|------|--------|
| **What happened** | Amazon's AI coding agent Kiro deleted a production environment |
| **Root cause** | Engineer had over-broad permissions; Kiro inherited them |
| **Impact** | AWS Cost Explorer down for 13 hours |
| **Amazon's take** | "User error, not AI error" |
| **Real lesson** | AI agents need separate, restricted permissions + human approval for destructive actions |
| **Fix** | Mandatory peer review, least privilege for AI, blast radius limits |

> **The takeaway:** AI coding tools are powerful, but giving them unrestricted production access is like giving a new intern the root password on day one. Talent isn't the issue — guardrails are.

---

## References

- [Amazon Official Response — AWS Service Outage](https://www.aboutamazon.com/news/aws/aws-service-outage-ai-bot-kiro)
- [Financial Times — AI Coding Tool Caused AWS Outage (via Engadget)](https://www.engadget.com/ai/13-hour-aws-outage-reportedly-caused-by-amazons-own-ai-tools-170930190.html)
- [The Register — Amazon's Kiro "Vibed Too Hard"](https://www.theregister.com/2026/02/20/amazon_denies_kiro_agentic_ai_behind_outage/)
- [The Decoder — Kiro Decided to Delete and Recreate](https://the-decoder.com/aws-ai-coding-tool-decided-to-delete-and-recreate-a-customer-facing-system-causing-13-hour-outage-report-says/)
- [Futurism — Amazon's AI Caused Multiple Outages](https://futurism.com/artificial-intelligence/amazon-ai-aws-outages)

---

*Found this useful? Follow [@techvijayforyou](https://instagram.com/techvijayforyou) for more system design and incident breakdowns!*

**Author:** Vijay
**Handle:** @techvijayforyou
**Topic:** Production Incident — AI Agent Safety in Production

---

#systemdesign #tech #ai #softwareengineer #coding #aws #productionincident #devops #programming #kiro #aitools #cloudcomputing #techreels #developer #debugging #microservices #techcareer #codinglife #devlife #artificialintelligence
