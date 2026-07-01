# Escalation Rules — Crystallized Spec
## ShipBuddy Customer Success FTE

Extracted from `triage_agent` and `escalation_agent` in `src/agent/ship_agents.py`.
This is the human-readable version of the routing logic.

---

## Rule Set 1: Immediate Escalation (Override Everything)

These triggers bypass all resolution logic. The triage agent routes directly to
`EscalationHandler` without attempting any answer.

| Trigger | Keywords / Conditions | Route to |
|---|---|---|
| Data loss | "missing orders", "disappeared", "lost data", "can't find orders" | `marcus.webb@shipbuddy.io` |
| Tracking swap | "wrong tracking", "tracking numbers mixed up", "wrong customer's label" | `marcus.webb@shipbuddy.io` |
| Security breach | "unauthorized access", "someone else's account", "credential leak" | `marcus.webb@shipbuddy.io` |
| Platform bug (unresolvable) | Engineering-level issue, not in known issues workarounds | `marcus.webb@shipbuddy.io` |
| Enterprise SLA breach | Enterprise customer + "SLA", "downtime", "breach", "not met" | `priya.nair@shipbuddy.io` |
| Legal threat | "lawyer", "legal action", "sue", "chargeback" | `priya.nair@shipbuddy.io` |
| GDPR / DPA | "GDPR", "DPA", "data deletion", "right to erasure", "data export" | `priya.nair@shipbuddy.io` |

**Priority set to:** `urgent`
**EscalationHandler response:** 2–3 sentences only — acknowledge, say specialist is on it, give SLA.

---

## Rule Set 2: Billing Escalation

Escalate **only** when the customer is disputing a charge, not just asking about one.

| Escalate | Do NOT escalate |
|---|---|
| "This charge is wrong / incorrect" | "What is this charge?" |
| "I was charged twice" | "How does the overage work?" |
| "I dispute this charge" | "How is DIM weight calculated?" |
| Refund request with amount > $100 | "Will I be charged immediately?" |
| Trial extension request | "How do I upgrade?" |
| Duplicate charge confirmed | "What's included in my plan?" |

**Route to:** `billing@shipbuddy.io`
**Priority:** `high`

---

## Rule Set 3: Churn Escalation

Only applies to Pro and Enterprise customers.

| Trigger | Route to |
|---|---|
| "I'm cancelling", "cancel my account" | `priya.nair@shipbuddy.io` |
| "Switching to [competitor]", "evaluating alternatives" | `priya.nair@shipbuddy.io` |
| "This isn't working for us anymore" (Pro/Enterprise) | `priya.nair@shipbuddy.io` |

**Note:** Starter and Growth cancellations are handled by the resolution agent (explain offboarding process). Only Pro/Enterprise triggers churn escalation.

**Priority:** `high`

---

## Rule Set 4: Human Requested

| Trigger | Route to |
|---|---|
| "Talk to a human" | `escalations@shipbuddy.io` |
| "Real person", "actual person" | `escalations@shipbuddy.io` |
| "Manager", "supervisor" | `escalations@shipbuddy.io` |
| "This bot isn't helping" | `escalations@shipbuddy.io` |

**Priority:** inherits from ticket priority

---

## Rule Set 5: Sentiment-Driven Escalation (Follow-up Turns)

Does not apply on turn 1. Evaluated on turns 2+ when prior sentiment is available.

| Sentiment trend | Action |
|---|---|
| `neutral → concerned → frustrated` | Agent acknowledges frustration, attempts one more resolution |
| `frustrated → angry` | Consider escalating even if issue is normally resolvable |
| `any → panicked` | Immediate human handoff |
| Sentiment unchanged after 2 resolution attempts | Escalate — 3-strike rule |

---

## Escalation Handler Output Rules

When `EscalationHandler` generates a response, it must:

1. **Acknowledge** the specific issue in one sentence (name what the customer said, not generic "I see you're having trouble")
2. **State** a human specialist is being brought in now
3. **Give the SLA** — based on customer plan:

| Plan | Human response SLA |
|---|---|
| Enterprise | 15 minutes |
| Pro | 1 hour |
| Growth | 4 hours |
| Starter | 24 hours |

**Never:**
- Attempt to resolve the issue
- Make promises about outcomes
- Say "unfortunately", "kindly", "please don't hesitate to reach out"
- Start with "Hello" or "Thank you for reaching out"

**Example output (data loss, Pro plan):**
> Your orders from Tuesday April 29th have been flagged and our engineering team is investigating now.
> A specialist will be in touch within 1 hour with an update.

---

## Escalation Contacts Reference

| Contact | Handles |
|---|---|
| `marcus.webb@shipbuddy.io` | Data loss, engineering bugs, tracking issues, security |
| `priya.nair@shipbuddy.io` | SLA breach, legal, GDPR/DPA, Pro/Enterprise churn |
| `billing@shipbuddy.io` | Billing disputes, refunds, duplicate charges |
| `escalations@shipbuddy.io` | Human-requested escalations (any type) |

---

## What the Agent Can NEVER Do

These are hardcoded prohibitions — not left to agent judgment:

- Approve refunds or credits
- Grant trial extensions
- Waive overages
- Promise specific bug fix dates (beyond what docs state)
- Confirm or deny a security breach
- Provide legal or compliance advice
- Volunteer that it is an AI (unless directly asked)
