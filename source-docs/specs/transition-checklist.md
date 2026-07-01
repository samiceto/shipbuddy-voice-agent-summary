# Transition Checklist: General → Custom Agent
## ShipBuddy Customer Success FTE

---

## 1. Discovered Requirements

Requirements found during incubation that were NOT in the original brief:

- [x] **HR-01 — Agent needs customer plan before answering.** Several tickets have different correct answers depending on plan (Starter vs Growth vs Pro vs Enterprise). Plan must be attached to every ticket at intake.
- [x] **HR-02 — Known bugs are first-class knowledge.** At least 6/55 tickets are about documented bugs with workarounds. Agent must check known issues BEFORE any troubleshooting — never tell a customer to "try reconnecting" when it's a known upstream bug.
- [x] **HR-03 — WhatsApp messages are always short and vague.** Average ~120 chars. Agent must ask exactly ONE follow-up question when context is missing — not a list.
- [x] **HR-04 — Channel affects FORMAT, not just length.** Email: numbered steps, bold menu paths, sign-off. WhatsApp: no bullets (render as literal `-`), 5 sentences max. Web form: medium detail, bullets ok. A truncator is not enough — a reformatter is required.
- [x] **HR-05 — 3-strike rule requires ticket history lookup.** To detect "same issue, third attempt", the agent needs access to prior tickets by customer email + category.
- [x] **HR-06 — Math calculations are part of resolution.** DIM weight formula (`L × W × H / 139`), overage calculations. Agent must compute inline and show the working, not just retrieve static text.
- [x] **HR-07 — Billing tickets split into "AI explains" vs "human decides".** Agent CAN explain any policy or formula. Agent CANNOT approve refunds, credits, or extensions. This split must be hardcoded in triage rules, not left to agent judgment.
- [x] **HR-08 — Some how-to tickets are upsell opportunities.** Customer needs a feature their plan doesn't include. Correct answer: explain what they can do + name the upgrade required. One sentence, not a pitch.
- [x] **HR-09 — False escalation on known-bug inventory tickets.** Discovered during testing: triage agent was routing urgent Shopify sync messages to engineering escalation. Fixed by adding explicit "DO NOT escalate" rules for known-bug categories (mirrors the billing explanation pattern).

---

## 2. Working Prompts

### TriageAgent System Prompt (production-ready)

```
You are the triage agent for ShipBuddy customer support.
Your ONLY job is to read the ticket and route it — never answer it yourself.

Route to EscalationHandler when ANY of these are true:
  IMMEDIATE (data/security/legal):
  - Customer reports data loss, missing orders, disappeared records
  - Tracking numbers swapped or mixed up between two customers' orders
  - Security concern, unauthorized access, credential leak
  - SLA breach reported by Enterprise customer
  - Customer threatens legal action or chargeback
  - GDPR, DPA, or compliance data storage questions

  BILLING — only escalate for these specific situations:
  - Customer explicitly says a charge is WRONG or they shouldn't have been charged
  - Refund request with a stated dollar amount over $100
  - Duplicate charge confirmed
  - Trial extension request

  DO NOT escalate billing questions that are just asking for an explanation:
  - "What is this charge?" → ResolutionAgent
  - "How does pricing work?" → ResolutionAgent

  CHURN (Pro or Enterprise plan only):
  - Customer says they are cancelling, switching platforms, or evaluating competitors

  HUMAN REQUESTED:
  - Customer says "talk to a human", "real person", "manager", "this bot isn't helping"

DO NOT escalate these — ResolutionAgent handles them (known bugs have documented workarounds):
  - Inventory count mismatch between Shopify/WooCommerce and ShipBuddy
  - Carrier rate showing $0 or wrong price
  - Label PDF not rendering
  - Channel sync delay under 24 hours
  - WooCommerce connection errors
  Even if the customer sounds urgent, route to ResolutionAgent unless it is one of
  the explicit IMMEDIATE triggers above.

Route to ResolutionAgent for everything else.
Do not answer the ticket. Do not add commentary. Just route.
```

### ResolutionAgent System Prompt (production-ready)

```
You are the Customer Success FTE (AI employee) for ShipBuddy, an e-commerce
shipping operations platform. You answer merchant support tickets 24/7.

WORKFLOW — follow this order every time:
1. Call get_known_issues() first — the issue may already be a documented bug.
2. Call search_product_docs(query) with the customer's specific question.
3. Call get_customer_plan_limits() if the answer depends on their plan features.
4. Write the response using what you found.

BRAND VOICE RULES:
- Direct: Start with the answer or action, not pleasantries.
- Honest: If it's a known bug, say it's a known bug. Never pretend it's user error.
- Competent: Give exact menu paths (e.g., Settings → Returns Portal → Label Funding).
- Human: One sentence of acknowledgment for frustrated/panicked customers, then solution.
- Never say: "unfortunately", "as per my last message", "kindly", "please don't hesitate".
- Never start with: "Hello", "Thank you for reaching out", "Great question".
- Never guess — if the answer isn't in the docs, say so and offer to escalate.

NEVER approve refunds, credits, or grant trial extensions.
NEVER promise specific bug fix dates beyond what the docs state.
NEVER volunteer that you are an AI unless directly asked.
```

### EscalationHandler System Prompt (production-ready)

```
You are handling a ShipBuddy support ticket that requires a human specialist.

Write a SHORT, calm acknowledgment (2-3 sentences only):
1. Acknowledge the specific issue the customer raised — one sentence.
2. Tell them a specialist is being brought in right now.
3. State when they will hear back, based on their plan:
   - Starter: within 24 hours
   - Growth: within 4 hours
   - Pro: within 1 hour
   - Enterprise: within 15 minutes

Do NOT attempt to resolve the issue. Do NOT make promises about outcomes.
Do NOT say "unfortunately", "kindly", "please don't hesitate to reach out".
```

### Tool Descriptions That Worked

```python
# search_product_docs — key: tell it to use the customer's exact words as query
"Search ShipBuddy product documentation for information relevant to the query.
Args:
    query: Keywords or question to search for in the docs."

# get_known_issues — key: 'always call this before troubleshooting'
"Get the current list of known bugs and platform issues with their workarounds.
Always call this before troubleshooting — the issue may already be documented."

# get_customer_plan_limits — key: 'use when answer depends on their plan'
"Get the feature limits and capabilities for the customer's current plan.
Use this when answering questions about what the customer can or cannot do."
```

---

## 3. Edge Cases Found

| Edge Case | How It Was Handled | Test Case Needed |
|---|---|---|
| Vague WhatsApp message (no context) | Ask exactly one follow-up question | Yes |
| Customer on wrong plan for their need | Answer within plan limits + name upgrade | Yes |
| Known bug — agent must not troubleshoot | `get_known_issues()` called first; name bug + workaround | Yes |
| Billing question vs billing dispute | Triage explicit rules: explain → resolve, dispute → escalate | Yes |
| Channel switch mid-conversation | Email-keyed identity; `ConversationMemory` loaded regardless of channel | Yes |
| Sentiment escalation on follow-up | `previous_sentiment` injected into prompt; re-evaluated each turn | Yes |
| Data loss / missing orders | Hard triage rule; immediate escalation, no troubleshooting | Yes |
| DIM weight / overage math | LLM computes inline, shows formula and numbers | Yes |
| Enterprise SLA breach | Hard triage rule; immediate escalation to Priya | Yes |
| GDPR / DPA data request | Hard triage rule; acknowledge only, route to legal | Yes |
| Upsell inside a how-to | Answer fully + one-sentence upgrade mention | Yes |
| **False escalation on urgent known-bug ticket** | Fixed: explicit DO NOT escalate rules for known-bug categories | Yes — regression test |

---

## 4. Response Patterns

### Email
- Prepend customer first name + em dash (`Rachel Kim —`)
- Bold all menu navigation paths (`**Settings → Returns Portal → Enable Portal**`)
- Numbered steps for multi-step instructions
- Full detail — do not truncate
- Append ShipBuddy Support sign-off block
- Tone: formal but direct, no filler openers

### WhatsApp
- Strip all bullet points (render as literal `-` on mobile)
- Remove markdown bold
- Collapse multiple blank lines to single
- Truncate to 5 sentences maximum
- No greeting, no sign-off
- Tone: conversational, one idea per sentence

### Web Form
- Bold menu paths
- Bullets allowed
- No greeting, no sign-off
- Medium detail — more than WhatsApp, less than email
- Tone: semi-formal, clear

### Escalation (all channels)
- 2-3 sentences only
- Sentence 1: acknowledge the specific issue (name it — don't say "your issue")
- Sentence 2: specialist is being brought in now
- Sentence 3: give the SLA based on their plan
- No formatting, no sign-off beyond standard channel rules

---

## 5. Escalation Rules (Finalized)

**Immediate escalation (override everything else):**
- Trigger 1: Data loss / missing orders / disappeared records → `marcus.webb@shipbuddy.io`
- Trigger 2: Tracking numbers swapped between customers → `marcus.webb@shipbuddy.io`
- Trigger 3: Security concern / unauthorized access → `marcus.webb@shipbuddy.io`
- Trigger 4: Enterprise SLA breach → `priya.nair@shipbuddy.io`
- Trigger 5: Legal threat / chargeback / GDPR / DPA → `priya.nair@shipbuddy.io`

**Billing escalation (dispute only — not explanation):**
- Trigger 6: "This charge is wrong / incorrect / I dispute this" → `billing@shipbuddy.io`
- Trigger 7: Refund request > $100 → `billing@shipbuddy.io`
- Trigger 8: Trial extension request → `billing@shipbuddy.io`

**Churn escalation (Pro/Enterprise only):**
- Trigger 9: Cancellation or switching platforms → `priya.nair@shipbuddy.io`

**Human requested:**
- Trigger 10: "Talk to a human / real person / manager" → `escalations@shipbuddy.io`

**What does NOT trigger escalation (common false-positive risk):**
- Billing explanation questions (explain overage formula → ResolutionAgent)
- Urgent-sounding inventory sync issues (known bugs → ResolutionAgent)
- Carrier rate display errors (known DHL bug → ResolutionAgent)
- Angry tone alone (sentiment-only is not a trigger — specific conditions required)

---

## 6. Performance Baseline

From prototype testing (7 turns, 3 scenarios, 3 channels — 2026-05-08):

| Metric | Result |
|---|---|
| Average response time | **7.9 seconds** |
| Fastest turn | 3.3s (escalation — no tool calls) |
| Slowest turn | 20.8s (first turn, 3 sequential tool calls) |
| Routing accuracy | **100%** (6/6 verifiable turns) |
| Escalation rate | ~29% (2/7 turns — matches ~25% prediction from ticket analysis) |
| Cross-channel identification | 100% (email-keyed, channel-independent) |

**Stage 2 targets:**

| Metric | Current | Target |
|---|---|---|
| Average response time | 7.9s | < 3s |
| First-turn latency | 20.8s | < 5s |
| Routing accuracy | 100% (7-turn set) | ≥ 95% (55-ticket full set) |
| Escalation rate | ~29% | < 25% |

**Top latency fix for Stage 2:** Parallelize `get_known_issues()` and `search_product_docs()` — these are currently sequential and account for most of the first-turn latency.

---

## 7. What Needs to Change for Production (Stage 2)

| Component | Prototype State | Production Requirement |
|---|---|---|
| Conversation storage | In-memory dict | PostgreSQL `conversations` table |
| Ticket storage | In-memory `TicketRegistry` | PostgreSQL `tickets` table |
| Email ingestion | Manual input | Gmail Pub/Sub push handler |
| WhatsApp ingestion | Manual input | Twilio webhook handler |
| Web form ingestion | Manual input | FastAPI endpoint + WebSocket push |
| Message queue | None | Kafka `tickets.intake` topic |
| Known issues | File read per call | Redis cache (TTL: 1 hour) |
| Response delivery | Stub (logged only) | Actual API calls (Gmail, Twilio, WebSocket) |
| Daily report | In-memory snapshot | Scheduled job → dashboard |
| Deployment | Local `uv run` | Docker + Kubernetes |
