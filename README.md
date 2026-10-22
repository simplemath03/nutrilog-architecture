# NutriLog: architecture and decision record

NutriLog is an AI nutrition and lifestyle coach that has been in production on Google Cloud since
June 2026. Users log meals by chat, photo, or barcode in a progressive web app; an LLM agent
reads and writes a PostgreSQL database through about 35 tools exposed over the Model Context
Protocol; every number the user sees comes from SQL, never from the model.

This repository holds the design documents and the decision log. **It contains no application
code.** The source is private because the app runs with real users' health data; I share it with
people evaluating my work on request.

- [ARCHITECTURE.md](ARCHITECTURE.md): the system design, component by component
- [DECISIONS.md](DECISIONS.md): selected entries from the decision log, in the order they were made
- This README: what it is, why it is shaped this way, and what the numbers say

## The problem it solves

A family member was tracking meals by chatting with a general-purpose assistant. The experience
was right and the data was wrong: totals drifted, history scattered across conversations, and
there was no way to ask "what did I eat last Tuesday" and trust the answer. The chat history was
being used as a database.

NutriLog keeps the conversational interface and puts a real database behind it. The model is a
stateless front end over tools; aggregation, totals, and trends are deterministic SQL.

## Architecture

```
iPhone / Android (installed PWA)
  React + TypeScript + Vite · chat, Today, History, Trends, Settings, Admin
        │  HTTPS + Firebase Auth ID token
        ▼
Cloud Run: agent-service (Python 3.12, FastAPI)
  · verifies the token, enforces the DB allowlist and subscription tier
  · Postgres-backed sessions; image upload to Cloud Storage
  · hand-written tool-calling loop; tool calls in a round run in parallel
  · house path: google-genai SDK on Vertex AI (Gemini Flash / Pro by tier)
  · BYOK path: LiteLLM for user-supplied Gemini / Anthropic / OpenAI keys
  · streams tokens and structured tool_result events to the PWA over SSE
        │  MCP (Streamable HTTP) · Cloud Run service-to-service OIDC
        │  X-User-Id header set by the agent, never by the client
        ▼
Cloud Run: mcp-server (Python 3.12, FastMCP)
  · ~35 tools: logging, search, edit, goals, trends, TDEE, saved meals,
    custom trackers, barcode lookup, export
  · pure CRUD and SQL aggregation; zero model-specific code
        │
        ▼
Cloud SQL (PostgreSQL 16)            Cloud Storage (private bucket, signed URLs)

Supporting: Firebase Auth and Hosting · Secret Manager · Cloud KMS · Cloud Scheduler ·
Artifact Registry · Terraform for all of it
```

Three shape decisions carry most of the weight:

1. **The MCP server is a separate service, not functions inlined in the agent.** The tool layer
   is model-agnostic and independently testable, and it is the piece that survives a model swap.
   The cost is one network hop.
2. **The agent sits between the client and the model.** The model key never reaches the browser,
   identity and tool authorization are enforced server-side, and the model is swappable behind one
   `ChatModel` interface. Two implementations exist behind that seam today, which is how the
   abstraction was proved.
3. **Every SQL statement in the MCP server filters by the user id from the request header**, never
   from tool arguments and never from model output. A test asserts this on every query.

## Decisions worth reading

The full log has 49 entries. These are the ones that changed the system the most. Dates are when
the decision was made; several reversed an earlier one.

**Streaming bypasses Firebase Hosting (June).** Long model turns went silent after about 60
seconds, the reply was lost, but the tool writes had already committed. The first fix was an SSE
heartbeat, on the theory that Hosting had an idle timeout. It did not; Hosting buffers proxied
responses and hard-caps them. The second fix was the right one: the PWA calls the agent's Cloud
Run URL directly with CORS, and a buffering middleware was replaced with a pure-ASGI one. Hosting
still serves the static app.

**A manual tool-calling loop instead of the SDK's automatic mode (June).** The UI needs
`tool_result` events mid-turn so the Today header updates as the model logs a meal; the SDK's
auto mode hides tool results until the turn ends. The manual loop also gives per-call usage
metering and a hard cap on calls per turn.

**An accuracy protocol for estimates (June, after the first week of real use).** Estimation order
is now fixed: anchor on the user's prior logs first (identical items must produce identical
macros, and the most recent log wins so corrections carry forward); branded or restaurant food
with no anchor must use a web lookup; only generic food may be estimated directly. A live
regression suite checks this on every release.

**Latency, round one (August).** Measured shape of a turn: two preamble tool calls before the
first token, every tool call in a round executed serially, and the model thinking at its deepest
setting before each call. Fixes: one preamble call that carries the whole context snapshot; tool
declarations cached per process; a round's calls run under `asyncio.gather`; thinking depth made
tunable with shallower defaults; four hot-path indexes. The accuracy regression still passed at
the shallow setting, so nothing was traded away.

**Latency, round two (August).** Volatile context (the clock and the live snapshot) moved out of
the system instruction into the user turn, so the rules and tool schema became a stable,
cacheable prefix. MCP clients cached per user with a TTL; web-lookup answers cached for 30 days.
Across both rounds, p95 response time went from about 80 seconds to about 43.

**BYO keys: Fernet, then KMS envelope encryption (June, then August).** User-supplied model keys
are validated with a cheap live call, then stored as ciphertext plus last four digits, never
logged or returned. pgcrypto was rejected because the key would leak into SQL logs. Round one
used a single Fernet data key in Secret Manager; round two wrapped a per-row data key with a
Cloud KMS key on 90-day rotation, with legacy rows re-wrapped lazily on next use.

**Telemetry was silently off for three weeks (September).** The per-turn telemetry line added in
August never reached Cloud Logging because neither service called `logging.basicConfig`; Python's
default level dropped it. Every latency claim for those weeks had to be reconstructed from the
usage table. The fix included persisting rounds, tool time, and cached tokens per turn in SQL so
latency, cost, and prompt-cache hit rate are queries rather than estimates.

**Barcode lookup had never worked in production (September).** A dependency was present in the
shared dev environment but not declared by the service's own package, so the container raised on
first use. The lesson recorded: a lazy import inside a tool body defers a packaging bug to
runtime. The same incident separated transport failures from genuine not-found results so an
upstream outage can never masquerade as "product isn't in the database." A unit test had already
caught a rounding error in the salt-to-sodium conversion.

**Enter stays a newline (August, reverted the same day).** A release made Enter send the message.
Users write meals in blocks. Reverted within hours; send is the button or Ctrl/Cmd+Enter.

## What the numbers say

| Measure | Value |
|---|---|
| In production since | June 11, 2026 |
| Releases | 8 (v0.2 through v0.8.2) |
| Tools exposed over MCP | ~35 |
| Automated tests across both services | 120+, plus live LLM regression suites for accuracy and coaching behavior |
| Versioned SQL migrations | 13 |
| p95 response latency | ~80 s to ~43 s across two optimization rounds |
| Failed turns in the three weeks after v0.8.2 | 0 |
| Monthly infrastructure cost at household scale | roughly $12 to $25 |

The user base is intentionally small and invite-only. The point of the project was never scale;
it was to build and operate a complete product, with billing, quotas, auth, push notifications,
an admin console, and a decision record, to the standard I would hold a team to.

## Stack

Python 3.12 · FastAPI · FastMCP · PostgreSQL 16 on Cloud SQL · React · TypeScript · Vite ·
Recharts · Vertex AI (Gemini) · LiteLLM · Firebase Auth and Hosting · Cloud Run · Cloud Storage ·
Secret Manager · Cloud KMS · Cloud Scheduler · Terraform · Stripe · Web Push (VAPID) ·
OpenFoodFacts

## How it was built

Solo, with Claude Code as the primary coding tool and the Gemini CLI for exercising the MCP
server directly. The planning document set (PRD, architecture, schema, implementation plan) was
written before the first line of code; the decision log was kept throughout. Every change ships
with tests; production behavior is measured with per-turn telemetry rather than assumed; two of
the entries above are incidents where the measurement caught something the tests did not.

## Source access

The application source is private. If you are evaluating my work and would like to read it, email
j.campbell@utexas.edu and I will add you as a read-only collaborator.
