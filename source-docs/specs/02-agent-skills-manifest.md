# Agent Skills Manifest
## ShipBuddy Customer Success FTE

This manifest documents the five reusable capabilities of the ShipBuddy Digital FTE.
Each skill maps to a concrete implementation in the codebase.

---

## Skill 1: Knowledge Retrieval

**Purpose:** Surface relevant product documentation and known bugs before generating any response.

| Field | Detail |
|---|---|
| **Trigger** | Every customer message that involves a product question, error, or how-to request |
| **Inputs** | `query: str` — keywords or question extracted from the customer message |
| **Outputs** | Ranked documentation snippets (top 3 by default), current known issues list |
| **Implementation** | `src/agent/tools.py` → `search_product_docs()`, `get_known_issues()` |
| **Called by** | `ResolutionAgent` (always calls both before drafting a response) |

**Behavior rules:**
- Known issues are checked **first** — if the customer's problem is a documented bug, say so immediately rather than asking them to troubleshoot.
- Doc search uses keyword overlap scoring against `##`-delimited sections of `context/product-docs.md`.
- If no docs match, the agent reports that clearly and offers to escalate — it does not guess.

**Example:**
```
Input:  "My Shopify inventory count doesn't match"
Output: [Known Issues section about Shopify webhook delay]
        [Docs section: Inventory Sync → Shopify → Force Sync]
```

---

## Skill 2: Sentiment Analysis

**Purpose:** Detect the customer's emotional state so the agent selects the correct tone before writing a response.

| Field | Detail |
|---|---|
| **Trigger** | Every incoming customer message, including follow-ups |
| **Inputs** | Customer message text + prior sentiment history (if follow-up turn) |
| **Outputs** | `detected_sentiment: str` — one of: `positive`, `curious`, `neutral`, `concerned`, `frustrated`, `angry`, `panicked` |
| **Implementation** | LLM-native — encoded in `ResolutionAgent` and `EscalationHandler` instructions in `src/agent/ship_agents.py` |
| **Stored in** | `ConversationMemory.sentiment_history[]` and `current_sentiment` (`src/agent/memory.py`) |

**Behavior rules:**
- Sentiment is assessed from the customer's words, not the agent's tone.
- On follow-up turns, if sentiment has worsened since the prior turn (e.g., `concerned → frustrated`), the agent must reflect that escalation in its response tone.
- Sentiment history is preserved across the full conversation and fed back into the agent context on every subsequent turn.

**Tone mapping:**

| Sentiment | Agent tone |
|---|---|
| positive / curious / neutral | Efficient, direct |
| concerned / frustrated | Calm, fast to the solution, one acknowledgment sentence |
| angry | Even, not defensive, de-escalate before solving |
| panicked | Steady, immediately action-oriented, no filler |

---

## Skill 3: Escalation Decision

**Purpose:** Determine whether a ticket should be routed to a human specialist, and identify who.

| Field | Detail |
|---|---|
| **Trigger** | At intake (triage), and again after sentiment worsens on a follow-up turn |
| **Inputs** | Customer message, detected category, customer plan, sentiment trend, turn number |
| **Outputs** | `should_escalate: bool`, `escalation_type: str`, `escalate_to: str`, `escalation_reason: str` |
| **Implementation** | `triage_agent` routing rules + `EscalationHandler` in `src/agent/ship_agents.py` |

**Hard escalation triggers (non-negotiable — triage routes immediately):**

| Trigger | Route to |
|---|---|
| Data loss, missing orders, disappeared records | `marcus.webb@shipbuddy.io` |
| Tracking numbers swapped between customers | `marcus.webb@shipbuddy.io` |
| Security breach, unauthorized access | `marcus.webb@shipbuddy.io` |
| SLA breach reported by Enterprise customer | `priya.nair@shipbuddy.io` |
| Legal threat, chargeback, GDPR/DPA request | `priya.nair@shipbuddy.io` |
| Pro/Enterprise customer cancelling | `priya.nair@shipbuddy.io` |
| Billing dispute (charge is wrong / refund > $100) | `billing@shipbuddy.io` |
| Customer explicitly requests a human | `escalations@shipbuddy.io` |

**What is NOT escalated (resolved by agent):**
- Billing questions that ask for an explanation ("what is this charge?")
- How-to questions, even complex ones
- Known bugs with documented workarounds
- Plan feature questions

**Human response SLA by plan:**

| Plan | SLA |
|---|---|
| Enterprise | 15 minutes |
| Pro | 1 hour |
| Growth | 4 hours |
| Starter | 24 hours |

---

## Skill 4: Channel Adaptation

**Purpose:** Reformat a resolved answer to match the conventions of the delivery channel before sending.

| Field | Detail |
|---|---|
| **Trigger** | After every agent response, before delivery |
| **Inputs** | `response: str`, `channel: "email" \| "whatsapp" \| "web_form"`, `customer_name: str` |
| **Outputs** | Channel-formatted response string |
| **Implementation** | `src/agent/formatter.py` → `format_for_channel()` |
| **Called by** | `agent.run()` (step 6), `fte_send_response` MCP tool |

**Per-channel rules:**

| Channel | Rules |
|---|---|
| `email` | Bold menu paths (`Settings → X → Y`), prepend customer name greeting, append ShipBuddy sign-off |
| `whatsapp` | Strip all bullet points (render as literal hyphens on mobile), remove markdown bold, max 5 sentences, collapse blank lines |
| `web_form` | Bold menu paths, keep bullets, medium detail |

**Why this matters:** A response generated for email (numbered steps, markdown, 400 words) sent to WhatsApp is unreadable. The formatter is not a truncator — it restructures the content for the medium.

---

## Skill 5: Customer Identification

**Purpose:** Identify the customer, load their conversation history, and detect channel switches before any agent logic runs.

| Field | Detail |
|---|---|
| **Trigger** | On every incoming message, before the agent pipeline starts |
| **Inputs** | `customer_email: str`, `customer_name: str`, `customer_plan: str`, `channel: str` |
| **Outputs** | `ConversationMemory` object (new or existing), `is_new: bool` |
| **Implementation** | `src/agent/memory.py` → `ConversationStore.get_or_create()` |
| **Called by** | `agent.run()` (step 1) |

**What it loads for returning customers:**
- Full OpenAI message history (passed directly to `Runner.run()` for continuity)
- Topics discussed in prior turns
- Sentiment trend (prior turns)
- Channel switches (e.g., email → WhatsApp)
- Resolution status of the ongoing conversation

**Channel switch detection:**
If `memory.current_channel != incoming_channel`, a `ChannelSwitch` event is recorded with a timestamp. The agent context is updated and the agent is aware it's handling a channel switch on this turn.

**Stage 2 note:** In production, `ConversationStore` is replaced by a PostgreSQL-backed repository with the same interface (`get_or_create`, `save`, `all_active`). No changes to the agent pipeline are needed.

---

## Summary Table

| Skill | Type | Implemented as | File |
|---|---|---|---|
| Knowledge Retrieval | Function tool | `search_product_docs()`, `get_known_issues()` | `src/agent/tools.py` |
| Sentiment Analysis | LLM-native | Agent instructions + structured output field | `src/agent/ship_agents.py` |
| Escalation Decision | Hybrid (rules + LLM) | `triage_agent` routing + `EscalationHandler` | `src/agent/ship_agents.py` |
| Channel Adaptation | Pure function | `format_for_channel()` | `src/agent/formatter.py` |
| Customer Identification | State lookup | `ConversationStore.get_or_create()` | `src/agent/memory.py` |

**Skill invocation order per message:**
```
Incoming message
      │
      ▼
[5] Customer Identification  ← load/create memory, detect channel switch
      │
      ▼
[2] Sentiment Analysis       ← LLM reads tone during triage
      │
      ▼
[3] Escalation Decision      ← triage_agent routes to correct handler
      │
      ▼
[1] Knowledge Retrieval      ← ResolutionAgent calls tools (if not escalating)
      │
      ▼
[4] Channel Adaptation       ← format_for_channel() before return
      │
      ▼
Formatted response
```
