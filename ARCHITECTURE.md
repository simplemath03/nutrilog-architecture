# NutriLog — System Architecture

Public copy of the internal architecture document. Production hostnames, project identifiers, and household details are replaced with placeholders; nothing else is changed. Section references to the PRD, schema, and implementation plan point at private documents.

---

## 1. Architecture Overview

Three deployable components plus managed data services, all on GCP:

```
┌─────────────────────────────────────────────────────────────────────┐
│ iPhone (Safari / installed PWA)                                     │
│   nutrilog-pwa: chat UI, photo capture, today dashboard             │
└──────────────┬──────────────────────────────────────────────────────┘
               │ HTTPS + Firebase Auth ID token (Authorization: Bearer)
               ▼
┌─────────────────────────────────────────────────────────────────────┐
│ Cloud Run: agent-service (Python, FastAPI)                          │
│  • Verifies Firebase ID token, enforces DB allowlist + tier         │
│  • Manages chat sessions (Postgres-backed)                          │
│  • Uploads images → Cloud Storage; passes bytes to Gemini           │
│  • House path: google-genai SDK (native manual tool loop);          │
│    BYOK path: LiteLLM SDK (user-supplied key)                       │
│  • CORS-enabled; PWA calls it DIRECTLY (not via Hosting /api)       │
│  • Streams model output to PWA via SSE                              │
└──────────────┬──────────────────────────────────────────────────────┘
               │ MCP (Streamable HTTP) + Cloud Run OIDC service auth
               │ Header: X-Nutrilog-User-Id (set by agent, never client)
               ▼
┌─────────────────────────────────────────────────────────────────────┐
│ Cloud Run: mcp-server (Python, FastMCP)                             │
│  • Tools: log_meal, log_exercise, log_weight, get_daily_summary,    │
│    search_entries, edit_entry, delete_entry, update_goals,          │
│    get_trends, get_goals, get_user_context                          │
│  • Pure CRUD + SQL aggregation; zero LLM code                       │
└──────────────┬──────────────────────────────────────────────────────┘
               │ Cloud SQL Python Connector (private IP or connector)
               ▼
┌──────────────────────────┐      ┌────────────────────────────────┐
│ Cloud SQL (PostgreSQL 16)│      │ Cloud Storage                  │
│ db: nutrilog             │      │ bucket: nutrilog-uploads       │
└──────────────────────────┘      └────────────────────────────────┘

Supporting services:
  Firebase Auth (Google Sign-In) · Firebase Hosting (PWA static assets)
  Secret Manager (Gemini API key, DB password) · Cloud Scheduler (nightly jobs)
  Artifact Registry (images) · Cloud Build or GitHub Actions (CI/CD)
```

### Why this shape

- **MCP server as a separate service** (rather than functions inlined in the agent): keeps the
  tool layer model-agnostic and independently testable (MCP Inspector, Gemini CLI, future
  Claude). This is the piece that survives a model swap or productization. The cost is one extra
  network hop, acceptable at this scale.
- **Agent service in the middle** (rather than PWA → Gemini directly): the Gemini API key never
  reaches the client; tool auth and user identity are enforced server-side; conversation/session
  logic is centralized; the model is swappable behind one interface.
- **Custom PWA frontend** is required, not optional: the consumer Gemini app does not support
  custom MCP/tool connectors. Only the Gemini API, Gemini CLI, and Gemini Enterprise do.
- **PWA → agent goes direct to Cloud Run, not through Firebase Hosting** (v0.4): the PWA's
  `VITE_API_BASE` is the agent's Cloud Run URL. Firebase Hosting buffers proxied responses and
  hard-caps them at ~60s, so it cannot stream SSE — long PRO turns were silently dropped (the
  reply was lost even though the tool writes had already persisted). Calling Cloud Run directly
  streams fine. Hosting still serves the static PWA; only API calls bypass it. This requires
  CORS on the agent (`CORSMiddleware` allowing `https://<app-domain>` and the
  `.firebaseapp.com` origin); auth stays a Firebase Bearer token (no cookies). The
  `firebase.json` `/api` rewrite remains as a legacy fallback.

## 2. Component Specifications

### 2.1 PWA (`nutrilog-pwa`)

- **Stack:** Vite + React + TypeScript. Tailwind for styling. `vite-plugin-pwa` for
  manifest/service worker.
- **Hosting:** Firebase Hosting (free tier, automatic TLS, plays well with Firebase Auth).
- **Auth:** Firebase Auth JS SDK, Google provider, `signInWithRedirect`. ⚠️ On iOS Safari
  16.1+, redirect sign-in breaks under third-party storage partitioning unless the auth
  handler is same-origin: set `authDomain` to the app's own Firebase Hosting domain (Hosting
  serves `/__/auth/*` from the same origin). Persist session; silent token refresh.
- **Screens:**
  1. **Chat** (default): message list, input bar, camera button (`<input type="file"
     accept="image/*" capture="environment">` and photo library), sticky Today header.
  2. **Today/History:** calendar-ish list of entries by day; entry detail with edit/delete
     (calls agent-service REST endpoints that wrap the same MCP tools — see §2.2).
  3. **Trends** (v0.4): Recharts charts of a chosen metric over a period; code-split so it
     stays out of the main bundle. Backed by `GET /trends`.
  4. **Settings:** goals display, export button, sign out; an "AI models" card to add/remove
     BYOK keys; and a "What the assistant remembers" panel that views/edits `profile_notes`.
  5. **Admin** (admins only): manage users and pending access requests (`AdminScreen`).
  6. **Request access** (`RequestAccessScreen`): shown to a verified Google user who is not yet
     on the allowlist.
- **Networking:** `POST /chat/messages` (multipart when image attached) → SSE stream of
  tokens + structured events (`tool_result` events let the UI render confirmation cards).
- **Offline:** Service worker caches shell; outgoing messages queued with retry. (Read-only
  cached Today data is a nice-to-have.)

### 2.2 Agent Service (`agent-service`)

- **Stack:** Python 3.12, FastAPI, `google-genai` SDK, `firebase-admin` (token verification),
  `mcp` client SDK / FastMCP client.
- **Endpoints:**
  - `POST /chat/sessions` — create session (one per day by default).
  - `POST /chat/sessions/{id}/messages` — send text and/or images; SSE response.
  - `GET  /dashboard/today` — non-LLM: proxies `get_daily_summary` MCP tool for the header/
    dashboard (fast path, no model invocation).
  - `GET  /entries?date=...`, `PATCH /entries/{id}`, `DELETE /entries/{id}` — non-LLM CRUD
    proxies for the History UI (wrap `search_entries`, `edit_entry`, `delete_entry`).
  - `GET  /trends?metric=&period=&activity_level=` — non-LLM: proxies the `get_trends` MCP
    tool for the Trends tab (see §2.1).
  - `GET  /export` — CSV/JSON export.
  - **Identity / onboarding / settings (v0.4):**
    - `GET  /me` — returns email, role, `subscription_tier`, `allowed_tiers`, `profile_notes`,
      and which BYOK provider keys are saved.
    - `PUT  /me/profile-notes` — user edits the assistant's standing memory (see §2.2 memory).
    - `PUT|DELETE /me/keys/{provider}` — add/remove a BYOK key (see §BYOK).
    - `POST /access-requests` — a verified-but-not-allowlisted user requests access.
    - **Admin router** (`admin` role only): `/admin/users` CRUD, `/admin/access-requests`
      approve/deny (approval sets a tier and inserts the allowlist row).
- **Model integration:** two `ChatModel` implementations behind one interface (§10):
  - **House path** (`llm/gemini.py`, the default): the `google-genai` SDK with a native manual
    tool loop. This path keeps Gemini 3.x thought-signature replay and Vertex grounding, neither
    of which survives LiteLLM. Multimodal input: pass uploaded image bytes inline (and store a
    copy in GCS, recording `gcs_uri` for the entry's attachment record).
  - **BYOK path** (`llm/litellm_model.py`): the LiteLLM SDK, used only when a turn runs on a
    user-supplied key. It mirrors the native manual tool loop (streamed tool-call fragment
    assembly). See §BYOK.
  - Web/local search grounding still always runs on the house Vertex account via a shared module
    (`llm/grounding.py`), so a BYOK turn never spends the user's key on search.
  - **Tiers & thinking config** (`llm/gemini.py`): `free` → Gemini Flash; `premium` → Gemini
    Pro with `thinking_level=HIGH`, temperature 0.6 (standard 0.4, BYOK 0.4). Numbers stay
    tool-derived, so temperature only affects phrasing/insight.
- **System prompt (maintained in repo, versioned):** defines persona (concise nutrition logging
  assistant), rules: always call tools to persist/read; never compute running totals yourself;
  always echo logged values with units and today's progress; mark photo estimates as
  approximate; ask one clarifying question only when an estimate would be wildly ambiguous;
  use `search_entries` + confirmation before destructive edits; current date/time and user
  timezone injected per request. Rule 7 is now PROACTIVE, brief, skippable coaching (never
  moralizing, never overriding a standing sensitivity).
- **Context injection (v0.4):** every turn calls the `get_user_context` MCP tool (a compact
  read-only snapshot: today's totals vs budget, latest weight + recent trend, current estimated
  TDEE, frequent recent meals) and renders it into the system prompt — `render_system_prompt`
  now takes a `context` arg and `system.md` has a RECENT CONTEXT block — so the model reasons
  over real data without extra round-trips.
- **User-editable memory:** `profile_notes` (the per-user standing instructions injected every
  turn, auto-maintained by the agent via prompt rule 12) is viewable/editable by the user in
  Settings (`GET` via `/me`, `PUT /me/profile-notes`). Rule 12 now also honors explicit
  "remember this" / "forget that" commands and briefly confirms saves.
- **Session memory:** messages stored in `chat_messages` table; on each turn send the last N
  (~20) turns. No vector store in V1.
- **Identity propagation:** after verifying the Firebase token, the service resolves
  `users.id` and sends it to the MCP server via `X-Nutrilog-User-Id`. Clients can never set it.
- **Usage metering:** log model, input/output token counts per call into `llm_usage`.

### 2.3 MCP Server (`mcp-server`)

- **Stack:** Python 3.12, FastMCP (Streamable HTTP transport), SQLAlchemy Core or asyncpg,
  Pydantic models for tool I/O.
- **Transport/Auth:** Streamable HTTP at `/mcp`. Cloud Run service requires authenticated
  invocation; only the agent service's service account has `roles/run.invoker`. Additionally
  validates `X-Nutrilog-User-Id` is present and exists.
- **Tools:** full specs in `04-MCP-IMPLEMENTATION-PLAN.md`. All tools are deterministic
  DB operations; no LLM calls inside the MCP server.
- **Statelessness:** no in-memory state; safe to scale 0→N.

### 2.4 Data Services

- **Cloud SQL PostgreSQL 16.** Smallest viable instance (e.g., `db-f1-micro`/shared-core or
  the sandbox/enterprise smallest tier), 10 GB SSD, automated backups + PITR (7 days),
  private IP preferred. Schema: `03-schema.sql`, applied via numbered migrations
  (plain SQL + a tiny migration runner, or Alembic).
  - *v0.4 tables* (migration `db/migrations/004_tiers_and_byok.sql`): `users.subscription_tier`
    + widened `role` check; `allowed_users` (the DB allowlist); `access_requests` (onboarding);
    `user_provider_keys` (BYOK ciphertext + last-4). The migration backfills the allowlist from
    existing users and the prior env list, and bootstraps the owner as `admin`.
  - *Cost-sensitive alternative (documented, not default):* Firestore would be ~$0 at this
    scale but pushes aggregation into application code and complicates trend queries; rejected
    for V1. If monthly cost matters more than SQL ergonomics, a serverless Postgres
    (e.g., Neon free tier) is schema-compatible — keep DSN configurable.
- **Cloud Storage** bucket `<uploads-bucket>`: uploaded photos at
  `users/{user_id}/uploads/{uuid}.jpg`, uniform bucket-level access, no public access,
  lifecycle rule optional (e.g., delete originals after 1 year), signed URLs (15 min) for
  viewing in the PWA.

### 2.5 Scheduled Jobs

- **Cloud Scheduler → Cloud Run job (or endpoint):**
  - `nightly-rollup` (00:30 user TZ): upsert `daily_summaries` rows for the prior day
    (idempotent; summaries are also computed live, this is a cache/consistency check).
  - `weekly-recap` (P2): generate recap message into the user's session.

## 3. Authentication & Authorization

| Hop | Mechanism |
|-----|-----------|
| PWA → Agent | Firebase Auth ID token (Google Sign-In). Agent verifies with `firebase-admin`, checks the email against the **`allowed_users` Cloud SQL table** (v0.4 — never the `users` table; removing a row revokes access), maps to `users` row (auto-provision on first login if allowlisted). The `ALLOWED_EMAILS` env var remains a transition fallback for one release, then is retired. |
| Agent → MCP | Cloud Run service-to-service OIDC (agent SA has `run.invoker` on mcp-server). `X-Nutrilog-User-Id` header carries tenant identity. |
| Agent → Gemini API | **Vertex AI with service-account auth (decided)** — billing consolidated in the project, no key management, paid-tier data-governance terms (no training on data). The `google-genai` SDK supports AI Studio API key behind the same flag if ever needed. |
| MCP → Cloud SQL | Cloud SQL Python Connector with IAM DB auth or password from Secret Manager. |
| Anyone → DB/bucket | No public access. |

Multi-tenancy enforcement rule: **every SQL statement in the MCP server filters by
`user_id = :uid`** taken from the header — never from tool arguments and never from model
output. Add a test that greps/asserts this on every query.

### 3.1 Tiers, roles & entitlements (v0.4)

- **Subscription tiers** (`users.subscription_tier` in `free | premium`): `free` → Gemini Flash,
  `premium` → Gemini Pro. **Roles** are `user | admin | developer`: `admin` and `developer`
  bypass tier checks; only `admin` manages users.
- **Downgrade, not reject** (`agent-service/.../entitlements.py`): if a turn requests a tier the
  user isn't entitled to, the request is downgraded and a `notice` SSE event is emitted — never
  a 403.

### 3.2 Invite-gated onboarding (v0.4)

A verified-but-not-allowlisted Google user gets a structured `403 {reason: not_invited,
request_status}` and a "request access" screen; the request lands in an `access_requests` table.
An admin approves (assigning a tier, which inserts the allowlist row) or denies. True self-serve
signup + payments are deferred until billing + quotas exist.

### 3.3 BYO API keys (BYOK) {#byok}

BYOK is a **capability for any signed-in user** (their key, their cost), **not** a subscription
tier. Keys are encrypted app-side with **Fernet** (`cryptography` lib); the data key lives in
Secret Manager as `BYOK_ENCRYPTION_KEY` (Terraform resource `nutrilog-byok-key`). pgcrypto was
rejected (the key would leak into SQL logs); a KMS-envelope upgrade is left open by the opaque
`BYTEA` column. On save a key is validated with a cheap live call; only ciphertext + last-4 are
stored, never logged or returned. Storage: `user_provider_keys` table. Validated providers:
Gemini AI Studio, Anthropic, OpenAI (others best-effort). BYOK turns run through
`llm/litellm_model.py` (§2.2). Dependency note: `litellm` hard-pins `python-dotenv==1.0.1`,
which conflicts with `fastmcp`'s `>=1.1.0` as a pip warning only — both work at runtime.

## 4. Data Flow Examples

### 4.1 Photo label log
1. PWA sends `POST /chat/sessions/{id}/messages` multipart: text "2 servings of this" + JPEG.
2. Agent verifies token → uploads JPEG to GCS → calls Gemini with [history, text, image] and
   MCP tools attached.
3. Model reads label, computes per-serving × 2, calls `log_meal(description="...",
   calories=380, protein_g=24, ..., estimate_source="label", attachment_gcs_uri=...)`.
4. MCP server inserts `food_entries` row (+ `attachments` row), returns the stored entry **and
   fresh daily totals** in the tool result.
5. Model emits confirmation text using returned numbers; agent streams it; PWA also updates the
   Today header from the `tool_result` event payload.

### 4.2 "Where am I today?"
Model calls `get_daily_summary(date="today")` → MCP runs SQL aggregate → model phrases it.
(The PWA header uses the non-LLM `GET /dashboard/today` fast path for the same data.)

### 4.3 Correction
"Actually that salad was a double portion" → model calls `search_entries(query="salad",
date="today", limit=3)` → identifies entry id → calls
`edit_entry(entry_id=..., scale_factor=2.0)` → confirmation with updated totals.

## 5. Environments & Configuration

- **Environments:** `dev` (local Docker Compose: Postgres + both services + Vite dev server;
  fake GCS via local dir or `fake-gcs-server`) and `prod` (GCP). A separate GCP `staging` is
  unnecessary for V1.
- **Config via env vars:** `DATABASE_URL`, `GEMINI_API_KEY` (secret), `GEMINI_MODEL`,
  `GEMINI_MODEL_PRO`, `MCP_SERVER_URL`, `ALLOWED_EMAILS` (legacy fallback, being retired),
  `GCS_BUCKET`, `FIREBASE_PROJECT_ID`, `LOG_LEVEL`. PWA build: `VITE_API_BASE` (the agent's
  Cloud Run URL — see §1).
- **Secrets:** Secret Manager, mounted into Cloud Run as env vars — including
  `BYOK_ENCRYPTION_KEY` (Fernet data key, Terraform `nutrilog-byok-key`).
- **Cloud Run request timeout:** 600s (10 min) as a ceiling against a hung streaming turn. An
  SSE turn holds a connection open only while generating; idle tabs cost nothing (Cloud Run
  scales to zero and bills only during active requests). Set by `deploy.ps1 --timeout 600`.

## 6. Infrastructure as Code & CI/CD

- **IaC:** Terraform in `/infra` (preferred) covering: project APIs, Cloud SQL instance+db+user,
  GCS bucket, Artifact Registry, two Cloud Run services + job, service accounts + IAM bindings,
  Cloud Scheduler, Secret Manager secrets (values set manually). A `gcloud`-script fallback is
  acceptable if Terraform overhead feels heavy for V1, but state the choice in the repo README.
- **CI/CD:** GitHub Actions: lint + tests on PR; on main: build images → Artifact Registry →
  `gcloud run deploy`. PWA: `firebase deploy --only hosting`.
- **Migrations:** run as a release step (Cloud Run job executing migration runner) before
  service deploy.

## 7. Observability

- **Logging:** structured JSON to Cloud Logging. Every tool invocation logged with tool name,
  user_id, latency, success/failure (never log full meal text at INFO if privacy-sensitive;
  DEBUG only).
- **Error reporting:** GCP Error Reporting picks up unhandled exceptions.
- **Metrics that matter:** end-to-end message latency, tool failure rate, Gemini token usage
  per day (from `llm_usage`), Cloud SQL connections.
- **Alerting (lightweight):** budget alert on the GCP project; uptime check on agent `/healthz`.

## 8. Cost Estimate (single household, prod)

| Item | Est. monthly |
|------|--------------|
| Cloud SQL smallest tier + 10 GB | ~$9–15 |
| Cloud Run ×2 + job (scale to zero, low traffic) | ~$0–3 |
| Cloud Storage (few GB photos) | < $1 |
| Firebase Hosting + Auth | $0 |
| Gemini API (flash-tier, ~30–60 multimodal msgs/day) | ~$1–5 |
| Secret Manager, Scheduler, egress | < $1 |
| **Total** | **~$12–25/mo** |

Largest lever if cost matters: replace Cloud SQL with serverless Postgres (→ ~$3–10/mo total).

## 9. Security & Privacy Notes

- Health-adjacent personal data: keep the footprint minimal. No analytics SDKs, no third-party
  trackers in the PWA.
- Photos may contain incidental personal content — bucket is private; access via short-lived
  signed URLs only.
- Soft-delete (`deleted_at`) for entries; a true "purge my data" admin script (P1) deletes a
  user's rows + GCS objects.
- Gemini API data handling: review Google's current API data-use terms during build; if
  required, route via Vertex AI for stronger data-governance posture (flag exists, §3).
- Prompt-injection surface is low (single trusted user, no web content ingestion), but tool
  layer still enforces: server-side user scoping, no raw-SQL tools, destructive ops require the
  specific entry id.

## 10. Model Portability (the "Claude later" path)

Everything model-specific lives in `agent-service/src/nutrilog_agent/llm/`. To swap models:
implement the same `ChatModel` interface (send messages+images+MCP tools, stream text +
tool events) for the new provider. MCP server, DB, PWA: zero changes. As of v0.4 there are two
implementations behind the seam — the native `gemini.py` (house path) and `litellm_model.py`
(BYOK path) — proving the abstraction. This is also the seam where users already choose models:
the chat composer's old Pro toggle is now a model selector cycling Flash → Pro (if entitled) →
each saved BYOK provider.
