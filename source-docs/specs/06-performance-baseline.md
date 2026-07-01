# Performance Baseline
## ShipBuddy Customer Success FTE — Stage 1

Measured: 2026-05-08
Model: gpt-4o (OpenAI Agents SDK)
Test: 7 turns across 3 scenarios, 3 channels

---

## Response Time

| Turn | Scenario | Channel | Elapsed |
|---|---|---|---|
| A-T1 | Inventory sync, turn 1 | email | 20.8s |
| A-T2 | Inventory sync, turn 2 (follow-up) | email | 6.0s |
| A-T3 | Inventory sync, turn 3 (channel switch) | whatsapp | 6.8s |
| B-T1 | Billing question, turn 1 | web_form | 6.4s |
| B-T2 | Billing follow-up, turn 2 | web_form | 8.8s |
| C-T1 | Data loss, turn 1 | email | 3.3s |
| C-T2 | Data loss, follow-up | email | 3.5s |

| Metric | Value |
|---|---|
| **Total (7 turns)** | 55.5s |
| **Average per turn** | 7.9s |
| **Fastest** | 3.3s (escalation — fewer tool calls) |
| **Slowest** | 20.8s (first turn, doc search + known issues) |

**Why turn 1 is slowest:** ResolutionAgent calls `get_known_issues()` + `search_product_docs()` + `get_customer_plan_limits()` — three sequential tool calls before generating a response. Follow-up turns skip the known issues check and run faster.

**Why escalation turns are fastest:** EscalationHandler makes no tool calls — it writes a 2-3 sentence acknowledgment directly from the prompt context.

---

## Routing Accuracy

Test set: 7 turns, 6 with verifiable expected routing (1 excluded — sentiment-only check)

| Turn | Expected | Actual | Result |
|---|---|---|---|
| A-T1 | resolve (known bug, has workaround) | escalate=False | PASS |
| A-T2 | resolve (follow-up, still known bug) | escalate=False | PASS |
| B-T1 | resolve (billing explanation, NOT dispute) | escalate=False | PASS |
| B-T2 | resolve (billing follow-up) | escalate=False | PASS |
| C-T1 | escalate (data loss — immediate) | escalate=True | PASS |
| C-T2 | escalate (data loss, still open) | escalate=True | PASS |

**Routing accuracy: 6/6 (100%)**

---

## Sentiment Detection

| Turn | Detected | Notes |
|---|---|---|
| A-T1 | angry | Customer had 2 oversells, urgent tone — reasonable |
| A-T2 | angry | Force sync failed, still broken — correct |
| A-T3 | panicked | 5 oversells, losing money — correct |
| B-T1 | concerned | Confused about charge — correct |
| B-T2 | curious | Follow-up question, calmer — correct |
| C-T1 | frustrated | Missing orders, needs records — correct |
| C-T2 | frustrated | 3 hours no response — slightly understated (could be angry) |

Sentiment detection is LLM-native and tracked in `ConversationMemory.sentiment_history`.

---

## Issues Found During Testing

### Issue 1: False escalation on known-bug inventory tickets (FIXED)
- **Before fix:** A-T1 and A-T2 escalated to engineering (routing accuracy: 4/6, 66%)
- **Root cause:** Triage agent treated urgent inventory sync messages as "engineering bugs" requiring escalation, rather than routing to ResolutionAgent which checks known issues first
- **Fix:** Added explicit "DO NOT escalate" rules to triage_agent for known-bug categories (Shopify sync delay, DHL rate display, Safari PDF, WooCommerce PHP 8.3)
- **After fix:** A-T1 and A-T2 resolve correctly (routing accuracy: 6/6, 100%)

---

## Stage 2 Targets

| Metric | Current | Target |
|---|---|---|
| Average response time | 7.9s | < 3s (streaming + DB cache) |
| Routing accuracy | 100% (6/6) | 95%+ on 55-ticket full set |
| First-turn latency | 20.8s | < 5s (parallel tool calls) |
| Escalation turn latency | 3.3s | < 2s |

**To hit the latency targets in Stage 2:**
- Run `get_known_issues()` and `search_product_docs()` in parallel (currently sequential)
- Cache known issues in Redis (changes rarely)
- Stream responses to channel instead of waiting for full output
