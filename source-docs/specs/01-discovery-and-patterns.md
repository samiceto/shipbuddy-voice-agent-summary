# Discovery Report: Customer Success AI Agent
## ShipBuddy — Incubation Phase Analysis

---

## 1. Ticket Volume & Channel Distribution

Analyzed 55 tickets across 3 channels:

| Channel | Count | Share | Message Length |
|---|---|---|---|
| Email | 29 | 53% | Long, detailed, complex |
| Web Form | 13 | 24% | Medium, structured |
| WhatsApp | 13 | 24% | Short, 77–156 chars average |

**Key finding:** Email is the dominant channel and carries the most complex, high-stakes tickets. WhatsApp messages are consistently short — the agent must handle incomplete context gracefully (e.g., no order number provided, vague description).

---

## 2. What Customers Actually Ask About

Ranked by frequency across all 55 tickets:

| Rank | Category | Count | Auto-Resolvable? |
|---|---|---|---|
| 1 | Billing (overages, upgrades, refunds, payment) | 9 | Partially — explain policy, escalate disputes |
| 2 | General inquiries (feature questions, plan limits) | 5 | Yes |
| 3 | How-to (bulk print, alerts, routing rules) | 4 | Yes |
| 4 | API / Webhooks | 4 | Yes (basic), escalate if unresolved |
| 5 | Carrier rates | 3 | Yes — most are known bugs or formula explanations |
| 5 | Returns portal | 3 | Yes (setup), escalate (404 bug) |
| 5 | Channel sync (Amazon, eBay) | 3 | Try once, escalate if persists |
| 5 | Account management (roles, warehouses) | 3 | Yes |
| 5 | Onboarding | 3 | Yes |
| 5 | Order routing | 3 | Yes (config help), escalate (rule not applying) |

**Key finding:** The top 3 categories (billing, general inquiries, how-to) represent 33% of all tickets and are almost entirely documentable. The agent can resolve these without any tool use beyond reading product docs.

---

## 3. Priority Distribution

| Priority | Count | What it means for the agent |
|---|---|---|
| Normal | 24 (44%) | Standard resolution, no rush |
| High | 15 (27%) | Revenue or operations impacted, respond fast |
| Urgent | 6 (11%) | Business is actively losing money right now |
| Low | 10 (18%) | Informational, can batch |

**Key finding:** 38% of tickets are high or urgent. The agent must triage priority at intake, not treat all tickets equally. An urgent ticket about 37 unfulfilled Amazon orders cannot wait in the same queue as "does ShipBuddy have a mobile app?"

---

## 4. Sentiment Landscape

Grouped into 4 buckets for agent tone selection:

| Bucket | Sentiments | Count | Agent Tone |
|---|---|---|---|
| **Calm** | neutral, positive, inquisitive, cautious | 23 | Efficient, friendly |
| **Distressed** | frustrated, concerned, worried | 15 | Calm, direct, fast to solution |
| **Panic mode** | panicked, alarmed, stressed | 5 | Steady, in control, immediate action |
| **Adversarial** | angry, firm, decided, professional_angry | 5 | Even, not defensive, escalate if needed |

**Key finding:** 45% of tickets come from customers who are already in some negative state. The agent's tone detection must happen BEFORE generating a response — a cheerful opener to a panicking customer is actively harmful.

---

## 5. Escalation Analysis

**75% of tickets (41/55) can be fully resolved by the AI.**  
**25% of tickets (14/55) require human handoff.**

### Tickets that must escalate:

| Ticket | Reason | Escalate To |
|---|---|---|
| TKT-015 | Cancellation, Pro plan customer | Priya Nair (churn risk) |
| TKT-021 | Billing dispute — overage count mismatch | Sofia Reyes |
| TKT-023 | Amazon tracking not uploaded, >1hr | Marcus Webb |
| TKT-028 | Trial expired — grace period extension | Jake Turner |
| TKT-031 | Tracking numbers swapped between orders | Marcus Webb (immediate) |
| TKT-034 | OnTrac ZIP coverage — engineering check | Marcus Webb |
| TKT-035 | eBay 2hr delay — abnormal, persistent | Marcus Webb |
| TKT-037 | Data loss — orders disappeared | Marcus Webb (immediate) |
| TKT-043 | Signature rule not applying — bug possible | Marcus Webb |
| TKT-044 | Returns portal 404 after slug verified | Marcus Webb |
| TKT-047 | Annual refund outside 30-day window | Sofia Reyes (exception) |
| TKT-050 | Enterprise SLA breach | Priya Nair + Legal (immediate) |
| TKT-051 | Carrier credit missing after 7 days | Sofia Reyes |
| TKT-055 | GDPR / DPA request | Priya Nair + Legal (immediate) |

### Escalation triggers the agent must detect:

1. **Keywords:** "data loss", "missing orders", "wrong tracking", "legal", "chargeback", "GDPR", "DPA", "SLA breach"
2. **Plan + action combo:** Customer is Pro/Enterprise AND is cancelling
3. **Billing amounts:** Any refund request or dispute > $100
4. **Persistence:** Same unresolved issue after agent's first resolution attempt
5. **Explicit request:** "talk to a human", "real person", "manager"

---

## 6. Hidden Requirements Discovered

These were NOT obvious from the project brief but emerged from ticket analysis:

### HR-01: The agent needs to know the customer's PLAN before answering
Several tickets have different correct answers depending on plan:
- TKT-054: Starter can only connect 1 channel — if they already have one, they need to upgrade
- TKT-024: ShipBuddy pre-negotiated rates only available on Growth+
- TKT-007: Real-time WooCommerce rates require plugin v2.4+ (beta)

**Implication:** The agent cannot just answer from docs — it needs customer context (plan, channels connected, customer_since date) attached to the ticket.

### HR-02: Known bugs are first-class knowledge, not edge cases
At least 6 of the 55 tickets are about KNOWN BUGS with documented workarounds:
- Shopify webhook delays (TKT-001)
- DHL $0 rate bug (TKT-004)
- Safari 17.4 PDF rendering (TKT-003)
- WooCommerce PHP 8.3 fatal error (TKT-010)
- Amazon SP-API 90-day token expiry (TKT-006, TKT-023)
- DHL international EasyPost upstream bug (TKT-004)

**Implication:** The agent needs a "Known Issues" knowledge layer that it checks FIRST before any troubleshooting. This prevents the agent from telling a customer to "try reconnecting" when the issue is a documented upstream bug.

### HR-03: WhatsApp customers give less context — agent must ask one targeted question
WhatsApp messages average ~120 characters. Customers don't attach order numbers, SKUs, or error messages. The agent must ask for exactly ONE missing piece of context in its first reply, not a list of questions.

### HR-04: Channel affects response FORMAT, not just LENGTH
- Email: numbered steps, bold menu paths, professional sign-off
- WhatsApp: no bullet points (renders as literal hyphens), 5 sentences max, line breaks only
- Web form: medium detail, hyperlinks acceptable

**Implication:** The agent needs a response formatter that takes a resolved answer and rewrites it to channel format — not just truncates it.

### HR-05: The 3-strike rule requires ticket history lookup
To enforce "escalate after 3 failed attempts", the agent needs access to previous tickets from the same customer. This means the database schema must support querying by customer email and issue category.

### HR-06: Math calculations are part of resolution
TKT-029: Customer thinks DIM weight is wrong. The correct answer requires computing `(12×10×8)/139 = 6.9 lbs`. The agent must be able to do simple arithmetic to explain carrier formulas — not just retrieve static text.

### HR-07: Billing tickets split into "AI explains" vs "human decides"
The agent CAN: explain policies, formulas, what a charge means.
The agent CANNOT: approve refunds, grant credits, extend trials, waive overages.
This split must be hardcoded — not left to the agent's judgment.

### HR-08: Some "how-to" tickets are actually upsell opportunities
TKT-041: Customer needs 3 warehouses. They're on Starter (1 warehouse max). Correct answer includes "you'll need to upgrade to Growth." The agent should frame this naturally, not as a sales pitch.

---

## 7. What the Agent Brain Needs

Based on all patterns above, the agent requires these capabilities:

```
INPUT: ticket (message + channel + customer metadata)
         │
         ▼
┌─────────────────────────────┐
│  1. TRIAGE                  │
│  - Detect priority level    │
│  - Detect sentiment bucket  │
│  - Flag immediate escalation│
│    keywords                 │
└────────────┬────────────────┘
             │
             ▼
┌─────────────────────────────┐
│  2. CONTEXT ENRICHMENT      │
│  - Look up customer plan    │
│  - Look up ticket history   │
│    (3-strike check)         │
│  - Check known issues list  │
└────────────┬────────────────┘
             │
             ▼
┌─────────────────────────────┐
│  3. DECISION                │
│  - Escalate immediately?    │
│  - Answer from docs?        │
│  - Ask for more info?       │
│  - Try fix + monitor?       │
└────────────┬────────────────┘
             │
             ▼
┌─────────────────────────────┐
│  4. RESPONSE GENERATION     │
│  - Select tone from         │
│    sentiment bucket         │
│  - Generate answer          │
│  - Format for channel       │
│    (email/whatsapp/web)     │
└────────────┬────────────────┘
             │
             ▼
┌─────────────────────────────┐
│  5. RECORD                  │
│  - Save ticket to DB        │
│  - Log resolution type      │
│  - Log channel metadata     │
│  - Emit daily report data   │
└─────────────────────────────┘
```

---

## 8. Database Schema Requirements (Derived from Tickets)

The following fields are required based on what the agent actually needs to answer tickets:

**customers table**
- id, name, email, company, plan, customer_since, channel_preference

**tickets table**
- id, customer_id, channel (email/whatsapp/web_form), subject, message
- category, priority, sentiment (detected)
- status (open / resolved / escalated_immediate / escalated_billing / escalated_engineering / escalated_churn / escalated_human_requested)
- resolution_type (auto_resolved / escalated)
- resolution_attempts (int — for 3-strike rule)
- created_at, resolved_at

**messages table**
- id, ticket_id, sender (customer / agent / human_csm), body, channel_format, created_at

**escalations table**
- id, ticket_id, reason, escalated_to (email address), escalated_at, status

---

## 9. Daily Report Requirements

The agent must generate a daily summary containing:
- Total tickets received (by channel)
- Resolution rate (auto vs escalated)
- Top 3 categories of the day
- Sentiment distribution (% negative)
- Urgent tickets handled
- Escalations triggered (count + reasons)
- Average response time per channel

---

## 10. System Architecture (High Level)

```
[Gmail] ──────────────────────────────────────────────┐
[WhatsApp/Twilio] ────────────────────────────────────┤
[Web Form/FastAPI] ───────────────────────────────────┤
                                                       ▼
                                              ┌─────────────────┐
                                              │   Kafka Topic   │
                                              │  tickets.intake │
                                              └────────┬────────┘
                                                       │
                                                       ▼
                                              ┌─────────────────┐
                                              │  Customer       │
                                              │  Success Agent  │
                                              │  (OpenAI SDK)   │
                                              └────────┬────────┘
                                          ┌────────────┼────────────┐
                                          ▼            ▼            ▼
                                    PostgreSQL    Escalation    Reply via
                                    (ticket DB)     Email      originating
                                                               channel
```

---

## 11. Open Questions for Next Phase

1. **Gmail polling vs Pub/Sub?** Polling is simpler, Pub/Sub is real-time. Given urgency of some tickets, Pub/Sub preferred.
2. **WhatsApp session handling:** Twilio sessions expire — how do we maintain conversation context across messages in the same ticket?
3. **Customer identification on WhatsApp:** Customer doesn't provide email in message. Match by phone number? Require them to provide email in first message?
4. **Web form → reply channel:** Customer submits form → do we reply by email? Do we show response in-browser? Both?
5. **Known issues freshness:** The known issues list will go stale. Who updates it and how often?
6. **Multi-language:** TKT-019 (French), TKT-021 (German company), TKT-042 (Spanish) — do we respond in customer's language?
