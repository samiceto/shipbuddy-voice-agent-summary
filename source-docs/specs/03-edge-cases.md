# Edge Cases & Handling Strategies
## ShipBuddy Customer Success FTE

Derived from analysis of 55 sample tickets and 8 hidden requirements discovered during incubation.

---

## EC-01: WhatsApp message with no context

**Description:** Customer sends a vague WhatsApp message with no order number, SKU, or error detail. WhatsApp messages average ~120 characters.

**Example:** `"my shipping isn't working"` or `"labels won't print"`

**Risk:** Agent tries to answer with generic advice that doesn't apply.

**Handling strategy:**
- Agent must ask for exactly **one** targeted follow-up question — not a list.
- Pick the single most useful missing piece (order number, carrier name, browser, etc.)
- Never ask more than one question in a WhatsApp turn.
- Formatted by channel adapter to max 5 sentences.

**Implementation:** WhatsApp constraint is in `formatter.py`. The resolution agent instructions say "never guess — if info is missing, ask one question."

---

## EC-02: Customer on wrong plan for their need

**Description:** Customer asks how to do something that requires a higher plan than they have.

**Example:** Customer on Starter asks to connect a second sales channel (Starter = 1 channel max). Customer on Growth asks about custom API rate limits (Enterprise only).

**Risk:** Agent either says "yes you can" (wrong) or just says "no" without offering a path forward.

**Handling strategy:**
- Answer what they can do on their current plan.
- Name the plan they'd need to upgrade to.
- State the upgrade path naturally, not as a sales pitch.
- Never approve the upgrade — that's a billing action.

**Implementation:** `get_customer_plan_limits()` tool in `tools.py` returns plan caps. Agent instructions say "frame upgrade naturally."

---

## EC-03: Known bug — agent must not troubleshoot

**Description:** Customer reports a problem that is a documented known bug (e.g., Shopify webhook delay, DHL $0 rate, Safari PDF rendering).

**Risk:** Agent tells customer to "try reconnecting" or "clear your cache" when the real answer is "this is a known bug with a workaround."

**Handling strategy:**
- `get_known_issues()` is called first, before any troubleshooting advice.
- If the issue matches a known bug: name the bug, state the workaround, give ETA if available.
- Never imply it might be user error when it's a documented platform bug.

**Known bugs currently tracked:**
- Shopify webhook delay (15–30 min during peak hours) — workaround: force sync
- DHL $0 rate display bug — workaround: use fallback carrier
- Safari 17.4 PDF rendering — workaround: use Chrome or download
- WooCommerce PHP 8.3 fatal error — workaround: downgrade to PHP 8.2
- Amazon SP-API 90-day token expiry — requires re-auth
- DHL international EasyPost upstream outage — no ETA, check status page

---

## EC-04: Billing question vs billing dispute

**Description:** Customer asks about a charge — but "asking about a charge" and "disputing a charge" require different responses.

**Risk:** Over-escalating (routing billing explanations to human) or under-escalating (agent approves a refund it shouldn't).

**Handling strategy:**

| Customer says | Agent does |
|---|---|
| "What is this $47.60 charge?" | Explain overage formula. No escalation. |
| "How does pricing work?" | Explain plan pricing. No escalation. |
| "This charge is wrong / I dispute this" | Escalate to `billing@shipbuddy.io` |
| "I was charged twice" | Escalate to `billing@shipbuddy.io` |
| "Refund > $100" | Escalate to `billing@shipbuddy.io` |
| "Can I get a trial extension?" | Escalate (cannot approve) |

**Implementation:** `triage_agent` has explicit "DO NOT escalate if just asking for explanation" rules.

---

## EC-05: Channel switch mid-conversation

**Description:** Customer starts on email, then sends a follow-up via WhatsApp (or web form). The agent must recognise this is the same conversation, not a new ticket.

**Risk:** Agent treats the WhatsApp message as a new customer and loses all prior context.

**Handling strategy:**
- Customer is identified by email, not by channel.
- `ConversationStore.get_or_create()` keyed on email — returns existing memory regardless of channel.
- A `ChannelSwitch` event is recorded with a timestamp.
- Agent context includes prior topics and sentiment trend.
- Response is formatted for the new channel.

**Implementation:** `agent.py` step 2 detects channel switch. `memory.py` stores `channel_switches[]`.

---

## EC-06: Sentiment escalation on follow-up turn

**Description:** Customer was "concerned" in turn 1, but is now "angry" or "panicked" in turn 2 because the first response didn't solve their problem.

**Risk:** Agent generates another calm resolution attempt when the customer needs a human.

**Handling strategy:**
- Sentiment history is injected into agent context on every follow-up turn.
- If sentiment trend is worsening (e.g., `concerned → frustrated → angry`), the agent must:
  1. Acknowledge the escalation explicitly.
  2. Consider escalating to human even if the issue was previously non-escalation.
- The triage agent re-evaluates escalation on each turn, not just turn 1.

**Implementation:** `_build_user_message()` in `agent.py` injects `previous_sentiment` into the prompt on follow-up turns.

---

## EC-07: Customer reports data loss or missing orders

**Description:** Customer says orders, records, or data have disappeared from ShipBuddy.

**Risk:** Agent attempts to troubleshoot a data integrity issue it cannot fix.

**Handling strategy:**
- Immediate escalation — no troubleshooting attempted.
- Routed to `marcus.webb@shipbuddy.io` (engineering).
- `EscalationHandler` writes a 2-3 sentence acknowledgment only.
- Priority set to `urgent` regardless of customer plan.

**Implementation:** Hard rule in `triage_agent` instructions: "missing orders, disappeared records" → `EscalationHandler` immediately.

---

## EC-08: Math-dependent answer (DIM weight, overage calculation)

**Description:** Correct answer requires arithmetic. Customer thinks their DIM weight or overage charge is wrong.

**Example:** "You charged me for 7 lbs but the box is 12×10×8 and only weighs 4 lbs."

**Risk:** Agent gives wrong number or refuses to engage with the calculation.

**Handling strategy:**
- Resolution agent must perform the calculation inline: `(L × W × H) / DIM_divisor`
- Standard DIM divisors: 139 (domestic), 139 (international varies by carrier)
- Show the formula and the customer's numbers, not just the result.
- If the calculated result differs from what was charged, flag for billing review.

**Implementation:** LLM-native arithmetic in agent response. No special tool needed for simple formulas.

---

## EC-09: Enterprise SLA breach

**Description:** Enterprise customer reports that ShipBuddy has violated their contracted 99.99% SLA.

**Risk:** Agent tries to resolve this technically when it's a contractual/legal matter.

**Handling strategy:**
- Immediate escalation to `priya.nair@shipbuddy.io`.
- Do not attempt to calculate uptime or verify the breach.
- Acknowledge and route — EscalationHandler writes the response.

---

## EC-10: GDPR / DPA data request

**Description:** Customer (typically EU-based) requests data export, deletion, or asks about data storage location.

**Risk:** Agent tries to answer GDPR questions from product docs — incorrect and potentially non-compliant.

**Handling strategy:**
- Immediate escalation to `priya.nair@shipbuddy.io` (legal).
- Agent writes acknowledgment only: "I've routed your data request to our compliance team."
- No data policies are cited from docs — legal must handle.

---

## EC-11: Upsell opportunity inside a how-to question

**Description:** Customer asks how to do something, and the answer reveals they need a higher plan.

**Example:** "How do I add a third warehouse?" (Starter plan: 1 warehouse max)

**Risk:** Agent just says "you can't" without explaining the path forward.

**Handling strategy:**
- Answer the how-to fully within the current plan limits.
- Name the plan upgrade required, and what it unlocks.
- One sentence — not a pitch.
- Example: "Starter supports one warehouse. Growth ($199/mo) gives you three."

---

## Summary

| Edge Case | Trigger | Handling | Escalates? |
|---|---|---|---|
| EC-01 | Vague WhatsApp | Ask one targeted question | No |
| EC-02 | Wrong plan for need | Answer + name upgrade path | No |
| EC-03 | Known bug | Name bug + workaround | No |
| EC-04 | Billing question vs dispute | Explain (question) / escalate (dispute) | Sometimes |
| EC-05 | Channel switch | Load prior memory, reformat | No |
| EC-06 | Worsening sentiment | Re-evaluate escalation each turn | Sometimes |
| EC-07 | Data loss / missing orders | Immediate escalation | Always |
| EC-08 | Math-dependent answer | Calculate inline, show formula | No |
| EC-09 | Enterprise SLA breach | Immediate escalation | Always |
| EC-10 | GDPR / DPA request | Immediate escalation | Always |
| EC-11 | Upsell in how-to | Answer + one-sentence upgrade mention | No |
