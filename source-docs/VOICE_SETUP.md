# ShipBuddy Voice Agent — Setup & Go-Live Guide

This service is the **brain** for live phone support: an OpenAI-compatible
Chat Completions endpoint that a managed voice platform (**Vapi**) calls as its
"Custom LLM". Vapi handles telephony, speech-to-text, text-to-speech, and
turn-taking; this server streams the answer.

```
caller speaks → Vapi (STT) → POST /vapi/chat/completions → voice agent (streamed)
            → Vapi (TTS) speaks the reply → caller hears it in ~1s
```

---

## Status: what's done vs. what's left

| # | Milestone | Status | Needs |
|---|-----------|--------|-------|
| 1 | Fork + repo + boots | ✅ done | — |
| 2 | Streaming endpoint (curl-testable) | ✅ done | — |
| 4 | Latency ~1s (warmup + inlined knowledge) | ✅ done | — |
| 5 | Escalation detection → handoff | ✅ done | — |
| 4b | Phone identity + cross-channel memory | ✅ done | — |
| 3 | **Wire Vapi → real phone call** | ⏳ remaining | **your accounts** (below) |

Everything code-only is complete and verified locally. The remaining work is
**account/deployment** steps that require your login, payment method, and a
public URL — see ["Remaining steps to go live"](#remaining-steps-to-go-live).

---

## Run & test locally (no accounts needed)

```bash
uv sync
uv run uvicorn src.voice.server:app --port 8080
```

The first boot does a one-time warmup (tens of seconds) so the first real caller
doesn't eat a cold start. Then test without any phone:

```bash
# normal answer (known DHL $0-rate bug)
curl -N http://localhost:8080/vapi/chat/completions \
  -H 'content-type: application/json' \
  -d '{"model":"shipbuddy-voice","stream":true,
       "messages":[{"role":"user","content":"my carrier rate shows $0, whats wrong?"}]}'

# escalation (data loss → human handoff)
curl -s http://localhost:8080/vapi/chat/completions \
  -H 'content-type: application/json' \
  -d '{"model":"shipbuddy-voice","stream":false,
       "messages":[{"role":"user","content":"all my orders disappeared!"}]}'

# known caller → cross-channel memory (recalls email/WhatsApp history)
curl -s http://localhost:8080/vapi/chat/completions \
  -H 'content-type: application/json' \
  -d '{"model":"shipbuddy-voice","stream":false,
       "call":{"customer":{"number":"+15551234567"}},
       "messages":[{"role":"user","content":"calling about my issue"}]}'
```

### Endpoints
- `POST /vapi/chat/completions` and `POST /chat/completions` — same handler
  (Vapi appends `/chat/completions` to the base URL you give it).
- `GET /health` — liveness check.

---

## Environment variables

| Var | Required | Default | Purpose |
|-----|----------|---------|---------|
| `OPENAI_API_KEY` | ✅ | — | The LLM brain. Loaded from `.env` with `override=True`, so `.env` wins over any stale key exported in the shell. |
| `VOICE_MODEL` | optional | `gpt-4o-mini` | Model on the hot (spoken) path. |
| `VOICE_ESCALATION_MODEL` | optional | `gpt-4o-mini` | Model for the escalation classifier. |
| `VOICE_SEED_DEMO` | optional | `1` | Seeds one known caller's email+WhatsApp history at boot for the cross-channel demo. Set `0` to disable. |
| `DATABASE_URL` | later | — | Only needed for **true** cross-process cross-channel memory (Stage 2 — see notes). |

> ⚠️ A stale `OPENAI_API_KEY` exported in your shell will shadow `.env` in other
> tools. This server forces `.env` to win; for other scripts, clear the export
> from your shell profile.

---

## Remaining steps to go live

These are the manual, account-bound steps. Order matters: deploy first (Vapi
needs a public URL), then Vapi, then a number.

### Step A — Deploy this server to a public URL  🙋 you (with my help on config)
Vapi must reach the endpoint over the internet; `localhost` won't work.

**⚠️ The existing `Dockerfile` and `render.yaml` deploy the _parent_ app
(`production.api.main:app`) and copy only `production/`.** For the **voice**
service you must run `src.voice.server:app` and include `src/` + `context/`.

Pick one (both are committed in this repo):

- **Render (simplest):** `render.yaml` already defines a second web service,
  **`shipbuddy-voice`**, that runs
  `uv run uvicorn src.voice.server:app --host 0.0.0.0 --port $PORT` with
  `healthCheckPath: /health`. Deploy the blueprint and set `OPENAI_API_KEY` in
  the dashboard (it's `sync: false`). The parent `shipbuddy-fte` service is
  untouched.

- **Docker (any host):** use **`Dockerfile.voice`** (separate from the parent
  `Dockerfile`). It installs deps, copies `src/` + `context/`, and serves on
  `$PORT` (default 8000):
  ```bash
  docker build -f Dockerfile.voice -t shipbuddy-voice .
  docker run -p 8000:8000 -e OPENAI_API_KEY=sk-... shipbuddy-voice
  ```

- **Quick demo without deploying:** run locally and expose it with a tunnel
  (`ngrok http 8080` or `cloudflared tunnel`). Use the public HTTPS URL it prints.

Confirm it's live: `curl https://<your-host>/health` → `{"status":"ok",...}`.

### Step B — Create a Vapi account  🙋 you (~5 min)
Sign up at vapi.ai, accept terms, copy your **API key**. (~$10 free trial credit —
enough to prove the live call works.)

### Step C — Create a Vapi Assistant with this as the Custom LLM  🙋 you / I can draft
In the Vapi dashboard create an Assistant and set its model to **Custom LLM**
pointing at your deployed base URL. The shape (confirm exact fields in the
current dashboard — Vapi's UI evolves):

```json
{
  "model": {
    "provider": "custom-llm",
    "url": "https://<your-host>",          // Vapi POSTs to {url}/chat/completions
    "model": "shipbuddy-voice"
  },
  "transcriber": { "provider": "deepgram", "model": "nova-2" },
  "voice":       { "provider": "cartesia" },
  "firstMessage": "ShipBuddy support, how can I help?"
}
```
- **STT/TTS:** Deepgram (STT) + Cartesia (TTS) are good defaults and have free
  credits. Vapi can also use its built-ins — no extra signups needed to start.
- The endpoint already streams OpenAI-style chunks, so Vapi gets first words fast.

### Step D — Attach a phone number  🙋 you (~5 min, needs payment method)
Either **import your existing Twilio number** into Vapi, or **buy one in Vapi**.
Assign the number to the Assistant from Step C.
> Telephony is the one unavoidable real cost — Twilio per-minute (~$0.0085/min).
> There's no free way to receive real PSTN calls.

### Step E — Call it  🎉
Dial the number. You should hear the agent answer, resolve known issues, and hand
off on escalation triggers — all with the ~1s latency.

---

## Optional / later enhancements (code-only, I can do these)

- **Recognize real callers:** add their phone → email/name/plan to the directory
  in `src/voice/identity.py` (`_DIRECTORY`). A known number taps that customer's
  cross-channel memory and uses their plan's SLA.
- **True cross-channel memory across services:** today the store is in-process,
  so voice shares memory within its own process and the seed demonstrates the
  link. Pointing both this service and the email/WhatsApp system at one
  **Postgres** (`DATABASE_URL`) makes the link real — `ConversationStore` keeps
  the same interface (Stage 2).
- **Live warm transfer to a human:** the agent currently speaks a handoff line
  and logs the escalation (with the routing contact). A real "ringing a person"
  transfer uses Vapi's call-transfer (`transferCall`) to a phone number — wired
  once Vapi + a number exist.

---

## How it behaves (verified locally)

- **Latency:** escalation path ~1.5s; normal answer ~1.6–3s under sustained
  traffic. (A slow *first* local call is the WSL2 cold-TLS reconnect artifact —
  it does not occur on a deployed host with steady traffic.)
- **Escalation:** data loss, "real person", disputed charges → human handoff;
  known bugs (DHL $0 rate, Safari label PDF, WooCommerce, inventory sync) are
  answered, not escalated.
- **Identity:** known number recalls email+WhatsApp history; unknown number is
  clean then recalls across a second separate call; the resolved plan drives the
  escalation callback SLA.
