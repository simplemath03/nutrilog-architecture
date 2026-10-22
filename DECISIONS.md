# NutriLog decision log, selected entries

The internal log has 49 entries, D1 through D49, kept from the first planning session on
June 11, 2026 through v0.8.2 in late September. This is a selection, lightly edited for
publication: production hostnames, project identifiers, and household details are removed;
the reasoning is unchanged. Numbering matches the internal log so the gaps are visible.

Where a later entry reverses an earlier one, both are kept. That is the point of the log.

---

**D2, LLM access: Vertex AI with service-account auth via the google-genai SDK.** Same per-token
price as a paid AI Studio key at expected volume, billing consolidated in the project, no API key
to manage, and paid-tier data terms (no training on user data), which matters for health data and
photos. The AI Studio path remains available behind the same SDK flag.

**D3, LLM strategy: Gemini first; Claude is the sanctioned fallback.** Pricing and accessibility
favor Gemini. If integration proves problematic, swap behind the `ChatModel` interface; the MCP
server, database, and PWA do not change. (As of v0.4 there are two implementations behind that
seam, the native Gemini path and a LiteLLM path for user-supplied keys, which is how the
abstraction was proved.)

**D4, Database: Cloud SQL PostgreSQL 16, smallest tier.** Stays on GCP, automated backups and
point-in-time recovery. Firestore was considered and rejected: it pushes aggregation into
application code and complicates trend queries. A serverless Postgres is documented as the cost
lever if monthly spend ever matters more than SQL ergonomics; the DSN is configurable.

**D5, Infrastructure as code: Terraform.** Reproducible; the whole project can be torn down and
recreated. (D12 later split responsibility: Terraform owns the foundation, a deploy script owns
releases, so Terraform does not churn on every image tag.)

**D8, A manual function-calling loop is the primary model integration, not the SDK's automatic
MCP mode.** The PWA contract requires streaming `tool_result` events mid-turn so the Today header
updates while the model works; the SDK's automatic mode hides tool results until the turn ends.
The manual loop also gives per-call usage metering and a hard cap on calls per turn.

**D16, The accuracy protocol (after the first week of real use).** Estimation order is fixed:
(1) anchor on the user's prior logs, where repeats collapse, the most recent log wins so
corrections carry forward, and identical items must produce identical macros; (2) branded,
packaged, or restaurant food with no anchor must use a web lookup; (3) only generic food may be
estimated directly. Portion ambiguity gets exactly one question, formatted as numbered options
the PWA renders as tappable chips. Durable user context (sensitivities such as "never mention
weight," tone, dietary identity) persists in editable profile notes rendered as overriding
standing instructions.

**D18, Production incident: Gemini 3.x on Vertex (June 12).** Every chat failed after the v0.3
rollout. Four root causes, all documented: 3.x models exist only on the global Vertex endpoint;
the pro tier's Vertex model ID differs from the public name; Maps grounding returns nothing on
the global endpoint so grounded sub-calls use a dedicated client pinned to a region; and Gemini
3.x requires replaying the original streamed parts, not reconstructed ones, or it returns 400.
Process fixes: local config now mirrors production model config so smoke tests cannot pass on a
stale model, and stream exceptions are logged with full tracebacks.

**D19, Typing lag was client-side, not a resource problem.** The image preview called
`URL.createObjectURL` inline in JSX, so every keystroke minted a new blob URL and re-decoded a
3 to 8 MB photo. Fix: create the object URL once at attach time. No cloud resources were changed
or needed. Photos are now downscaled client-side before upload (about 10x smaller, label text
still legible).

**D20b then D26, Long turns were dropped, twice diagnosed.** Symptom: no reply, but a reload shows
the meal saved. First diagnosis: Firebase Hosting's roughly 60-second idle timeout on the API
rewrite. Fix: a 15-second SSE heartbeat. Second diagnosis, same session: it was not an idle
timeout. Hosting buffers proxied responses, hard-caps them, and does not stream SSE at all. Real
fix: the PWA calls the agent's Cloud Run URL directly, the agent gains CORS for the Hosting
origins with bearer-token auth and no cookies, and a buffering middleware was rewritten as pure
ASGI. Hosting still serves the static app. The heartbeat entry is kept as a record of the wrong
first answer.

**D21, Subscription tiers and the allowlist live in Cloud SQL.** Firestore rejected for this:
split identity store, relational quota checks against the usage table, and no cost win unless
Cloud SQL is fully replaced. Entitlement policy is downgrade, not reject: a request for a tier
the user is not entitled to is served at the entitled tier with a notice event, never a 403.

**D22 then D35, BYO model keys: Fernet, then KMS envelope encryption.** Keys are validated with a
cheap live call on save and stored as ciphertext plus last four, never logged or returned.
pgcrypto was rejected because the key would leak into SQL logs; per-user Secret Manager rejected
on round-trip cost. Round one used a single Fernet data key held in Secret Manager; the opaque
`BYTEA` column was chosen so a KMS upgrade would be schema-compatible. Round two (D35) wrapped a
per-row data key with a Cloud KMS key on 90-day rotation; legacy rows still decrypt and are
re-wrapped lazily on next use; unwrapped keys are cached in-process for five minutes.

**D25, Trend charts come from the same aggregates the agent uses.** The in-app Trends tab reads
the same `get_trends` tool output as the model, so charts and the assistant always agree.
BigQuery and Looker Studio were rejected for per-user visualization: pipeline cost, a second copy
of private health data, and no secure consumer embedding.

**D30, Structured coaching style instead of free text.** One prompt behavior for every user
optimized for hitting numbers and produced supplement-style "fixes." Two deliberate changes:
a structured per-user `coaching_style` (strict, balanced, flexible) rendered as a behavior block
every turn, chosen over free-text notes because the UI can render it as a control and the model
cannot lose it in a bad notes merge; and a style-independent "real-world eating" rule (real
dishes in normal portions, targets are guides not walls, judge the week). The seven-day rollup
now rides in the context so "the week" is data in the prompt rather than a vibe.

**D31, Latency, round one.** Measured shape of a turn: two preamble tool calls plus a tool-list
fetch before the first token; every tool call in a round executed serially (the accuracy protocol
is three calls per item); the model thinking at its deepest setting before each call. Fixes: one
preamble call carrying the full context snapshot; tool declarations cached per process; a
round's calls run under `asyncio.gather` (verified the MCP session multiplexes concurrent
requests); thinking depth made tunable with shallower defaults; four hot-path indexes; a
structured `turn done` log line so future slowness can be attributed. The accuracy regression
still passed at the shallow setting, so the protocol was not relaxed. Rejected for now: a
per-user MCP client cache (lifecycle complexity on Cloud Run).

**D33, Update prompt and error boundary.** The PWA's service worker moved from auto-update to a
prompt, because a worker taking over mid-session could invalidate a hashed lazy chunk and blank
the screen with no recovery. An error boundary recognizes that failure and offers reload.

**D37, Token quotas.** Monthly allowance per tier computed from the usage table, with per-user
admin override. Exceeding it returns a structured error event before the turn starts, so nothing
is logged and the UI shows one clear message. Chosen over hard HTTP 402s.

**D40, Latency, round two.** Volatile context (the clock and the live snapshot) moved from the
system instruction into the user turn, so the rules and tool schema became a stable, cacheable
prefix. Per-user cached MCP client with a TTL for the non-model endpoints; identity cached 30
seconds and invalidated by the tools that write users; the DB pool pre-opened in the server
lifespan; web-lookup answers cached 30 days; several queries rewritten to be index-usable.
Across D31 and D40, p95 response time went from about 80 seconds to about 43.

**D43, Flexible style allows one affirming clause.** The model reliably appended a short
affirming remark after a log that landed over target, even when told the confirmation was the
whole reply. That is not nagging, so the contract was made honest instead of fought: flexible
means never a warning and never a suggestion to compensate, at most one affirming clause; strict
must name the overage and give one adjustment. A live test checks exactly that.

**D44, Enter stays a newline.** A release made Enter send and Shift+Enter a newline. Users write
meals in blocks, so this broke the primary input. Reverted the same day.

**D45, Telemetry was silently off for three weeks.** The `turn done` telemetry line from D31
never reached Cloud Logging because neither service called `logging.basicConfig`; Python's
default WARNING level dropped it. All latency analysis for those weeks had to fall back to the
usage table. Fix: both services configure logging at startup, and a migration persists rounds,
tool time, and cached tokens per turn so latency, cost, and prompt-cache hit rate are SQL queries
rather than estimates. Quota defaults were then re-sized from observed usage.

**D48, Barcode-first for packaged food.** A barcode lookup tool queries OpenFoodFacts directly
(free, keyless) instead of a web-lookup round-trip: exact label macros, about one second instead
of a second model call, cached 90 days. Salt-to-sodium conversion happens on raw grams, because
rounding first turned 0.125 g of salt into 40 mg instead of 50 mg; a unit test caught it.

**D49, Production incident: the barcode tool had never worked (September 21).** Two same-day
user reports, both root-caused from production evidence. (1) An HTTP client library was present
in the shared dev environment but not declared by the MCP server's own package, so the container
raised `ModuleNotFoundError` on first use. Lesson recorded: a lazy import inside a tool body
defers a packaging bug to runtime; dependencies used by tools are declared explicitly and imported
at module top. Transport and server errors are now separated from a genuine not-found so an
upstream outage can never masquerade as "product isn't in the database." (2) The push-notification
toggle appeared to drift off on its own. The browser was telling the truth: one-time notification
grants on Android revert when the user leaves the site and invalidate the subscription. The toggle
now stays device-truth, remembers that push was previously on, and explains the specific cause
with fix steps instead of a silently-off switch.
