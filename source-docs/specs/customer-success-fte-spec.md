# Customer Success FTE Specification
## ShipBuddy — Digital FTE (Incubation → Production)

---

## Purpose

Handle routine ShipBuddy merchant support queries with speed and consistency across email, WhatsApp, and web form — 24/7, with no human required for 75–80% of tickets. For the remaining 20–25%, collect context and route to the correct human specialist within SLA.

---

## Supported Channels

| Channel | Identifier | Response Style | Length |
|---|---|---|---|
| Email (Gmail) | Email address | Formal, detailed, markdown allowed | 500 words max |
| WhatsApp (Twilio) | Phone → resolved to email | Conversational, plain text only | 5 sentences max |
| Web Form (FastAPI) | Email address | Semi-formal, bullets allowed | 300 words max |

**Channel adaptation** is applied automatically by `format_for_channel()` before every response is sent. The agent generates one canonical response; the formatter rewrites it per channel (strip bullets for WhatsApp, bold menu paths for email/web, add sign-off for email).

**Cross-channel identity:** Customers are identified by email address across all channels. A customer who starts on email and follows up on WhatsApp continues the same conversation — no context is lost.

---

## Scope

### In Scope (agent handles autonomously)

- Product feature questions and plan comparisons
- How-to guidance (step-by-step with exact menu paths)
- Known bug acknowledgment + documented workaround delivery
- Billing policy explanations and overage formula calculations
- Returns portal setup and configuration
- Inventory sync troubleshooting (Shopify, WooCommerce, Amazon, eBay)
- API/webhook configuration questions
- Carrier rate explanations and DIM weight calculations
- Account management (roles, warehouses within plan limits)
- Onboarding questions
- Cross-channel conversation continuity

### Out of Scope (escalate to human)

| Trigger | Escalate to | SLA |
|---|---|---|
| Data loss / missing orders | `marcus.webb@shipbuddy.io` | Immediate |
| Tracking numbers swapped | `marcus.webb@shipbuddy.io` | Immediate |
| Security / unauthorized access | `marcus.webb@shipbuddy.io` | Immediate |
| Unresolvable engineering bug | `marcus.webb@shipbuddy.io` | Immediate |
| Billing dispute (charge is wrong) | `billing@shipbuddy.io` | Per plan SLA |
| Refund > $100 | `billing@shipbuddy.io` | Per plan SLA |
| Trial extension request | `billing@shipbuddy.io` | Per plan SLA |
| Enterprise SLA breach | `priya.nair@shipbuddy.io` | Immediate |
| Legal threat / chargeback | `priya.nair@shipbuddy.io` | Immediate |
| GDPR / DPA data request | `priya.nair@shipbuddy.io` | Immediate |
| Pro/Enterprise customer cancelling | `priya.nair@shipbuddy.io` | Per plan SLA |
| Customer requests a human | `escalations@shipbuddy.io` | Per plan SLA |

**Note on sentiment:** Escalation is triggered by specific conditions (data loss, legal, billing dispute), not by a raw sentiment score. Angry/panicked customers who have a resolvable issue are handled by the resolution agent with adjusted tone — they are not automatically escalated unless a hard trigger also applies.

---

## Agent Architecture

```
Incoming message
      │
      ▼
  TriageAgent          ← routing only, no answers
  ├── ResolutionAgent  ← answers, uses tools, formats response
  └── EscalationAgent ← 2-3 sentence acknowledgment, routes to human
```

**TriageAgent** reads the ticket and routes via handoff. It never generates a customer-facing response.

**ResolutionAgent** calls tools in order: `get_known_issues()` → `search_product_docs()` → `get_customer_plan_limits()` → draft response.

**EscalationAgent** writes a short acknowledgment only. It never attempts to solve the issue.

---

## Tools

| Tool | Purpose | Constraints |
|---|---|---|
| `fte_handle_ticket` | Run full agent pipeline for one message | Primary MCP entry point |
| `fte_search_knowledge_base` | Search product docs + known issues | Max 8 results, min 2-char query |
| `fte_create_ticket` | Log a support interaction | Required; include channel |
| `fte_get_customer_history` | Load prior tickets + conversation memory | Keyed by customer email |
| `fte_escalate_to_human` | Route ticket to specialist | Requires valid ticket_id |
| `fte_send_response` | Deliver formatted response via channel | Channel formatting applied automatically |

**Internal tools** (used by ResolutionAgent directly, not via MCP):

| Tool | Purpose |
|---|---|
| `search_product_docs(query)` | Keyword search over product-docs.md |
| `get_known_issues()` | Return current known bugs + workarounds |
| `get_customer_plan_limits()` | Return feature caps for customer's plan |

---

## Conversation Memory

Each customer has a `ConversationMemory` object that persists across turns:

| Field | Purpose |
|---|---|
| `messages` | Full OpenAI message history (passed to Runner for continuity) |
| `turn_count` | Number of turns in this conversation |
| `original_channel` / `current_channel` | Channel tracking |
| `channel_switches[]` | Timestamped log of channel changes |
| `topics_discussed[]` | Categories discussed (no duplicates) |
| `sentiment_history[]` | Per-turn sentiment, oldest to newest |
| `current_sentiment` | Latest detected sentiment |
| `resolution_status` | open → pending → solved / escalated |

**Stage 1:** In-memory dict keyed by customer email (`ConversationStore`).
**Stage 2:** Replaced by PostgreSQL-backed repository with identical interface.

---

## Performance Requirements

| Metric | Stage 1 Baseline | Stage 2 Target |
|---|---|---|
| Avg response time (processing) | 7.9s | < 3s |
| First-turn latency | 20.8s | < 5s |
| Escalation-turn latency | 3.3s | < 2s |
| Routing accuracy | 100% (6/6 test turns) | ≥ 95% on 55-ticket full set |
| Escalation rate | ~25% (from ticket analysis) | < 25% |
| Cross-channel identification | 100% (email-keyed) | ≥ 95% |

**Stage 2 latency improvements:**
- Run `get_known_issues()` and `search_product_docs()` in parallel (currently sequential)
- Cache known issues in Redis (TTL: 1 hour — changes rarely)
- Stream responses to channel instead of waiting for full completion

---

## Guardrails

| Rule | Enforced by |
|---|---|
| NEVER approve refunds, credits, trial extensions | Hard instruction in ResolutionAgent |
| NEVER promise specific bug fix dates not in docs | Hard instruction in ResolutionAgent |
| NEVER volunteer that the agent is an AI (unless asked directly) | Hard instruction in ResolutionAgent |
| NEVER say "unfortunately", "kindly", "please don't hesitate" | Brand voice rules in both agents |
| NEVER start with "Hello", "Thank you for reaching out", "Great question" | Brand voice rules |
| NEVER discuss competitor products | Omitted from agent knowledge entirely |
| ALWAYS check known issues before troubleshooting | Enforced by ResolutionAgent workflow order |
| ALWAYS format response for channel before delivery | Enforced by `format_for_channel()` in `agent.run()` |
| ALWAYS detect sentiment before drafting response | Structured output field `detected_sentiment` required |
| EscalationAgent NEVER attempts to resolve — acknowledge only | Hard instruction in EscalationAgent |

---

## Human SLA by Plan

| Plan | Human response SLA | Escalation contact |
|---|---|---|
| Enterprise | 15 minutes | per escalation_type routing |
| Pro | 1 hour | per escalation_type routing |
| Growth | 4 hours | per escalation_type routing |
| Starter | 24 hours | per escalation_type routing |

---

## Stage 2 Transition

The following components are stubs in Stage 1 and will be replaced in Stage 2:

| Component | Stage 1 | Stage 2 |
|---|---|---|
| Conversation storage | In-memory dict | PostgreSQL (`conversations` table) |
| Ticket storage | In-memory `TicketRegistry` | PostgreSQL (`tickets` table) |
| Email channel | Stub | Gmail API (Pub/Sub push) |
| WhatsApp channel | Stub | Twilio WhatsApp API |
| Web form channel | Stub | WebSocket push (Next.js) |
| Message queue | None | Kafka (`tickets.intake` topic) |
| Known issues cache | File read on every call | Redis (TTL: 1 hour) |
| Daily report | `store.report_snapshot()` dict | Scheduled job → dashboard |

See `specs/transition-checklist.md` for the full readiness checklist.
