# Repos & Conventions

All repos live side by side under `~/caliban/`. Each one is its own git repository, published in the [thecalibanproject](https://github.com/thecalibanproject) GitHub organization.

| Repo | Language | What it is |
|---|---|---|
| `docs/` | Markdown | Research, architecture and decisions (this repo). |
| `core/` | Rust (workspace) | Data plane (`caliban router`), control plane (`caliban control-plane`), and `caliban standalone` (both in one process, for on-prem). Owns the **API contract** (`core/api/openapi.yaml`) and the **config/ontology/node schemas** (`core/schemas/`). |
| `web/` | TypeScript, React, Vite | Admin console: tenants, BYOK keys, models, datasources and ontology studio, nodes, usage. Built to static files and served by the control plane. **No CDN, no external fonts, no telemetry.** |
| `sdk-typescript/` | TypeScript | `@caliban/sdk`: OpenAI-compatible client helpers, the Caliban extension fields, the admin API client, and node authoring types. |
| `sdk-python/` | Python | `caliban-sdk`: the same surface in Python, plus an eval harness for nodes and golden sets. |
| `ml/` | Python | Offline training and export: intent/router heads, PII NER evaluation and fine-tuning, BYO-model probe sets. Its only output to `core` is **ONNX + JSON** artifacts. |
| `deploy/` | Compose, Helm, shell | On-prem first: docker-compose stack, Helm chart, air-gap bundle scripts, example configs. |
| `website/` | Astro (static) | Product and architecture site. Renders the notes in `docs/` and the API contract from `core/` at build time. |

## Product decisions (2026-10-02)
- **BYOK only.** Customers bring their own provider keys, or their own local model endpoints. Caliban never pools upstream credentials. This also removes the KeyPooling cross-tenant cache leak by construction.
- **100% on-prem capable.** Every component, including the control plane, the web UI and every model, must run with **zero egress**. SaaS is the same software run by us.
- **Licensing (2026-10-03):** `sdk-typescript` and `sdk-python` are Apache-2.0; `core`, `web`, `ml`, `deploy`, `docs` and `website` are proprietary and source-available (public for reference; all rights reserved, use only under written agreement). Copyright 2026 Elie Sfeir. The PII NER model's CC-BY-SA fine-tuning-data item was signed off by Elie Sfeir on 2026-10-03 (recorded in the artifact manifest).
- Dependencies must be licensed for on-prem redistribution: Postgres, Qdrant (Apache-2.0), Valkey (BSD) and DataFusion (Apache-2.0) are fine. SSPL, BSL and non-commercial weights are not allowed in the default bundle.

## Runtime conventions

| Thing | Value |
|---|---|
| Data plane port | `8080`: OpenAI-compatible `/v1/*`, plus `/healthz` and `/metrics` |
| Control plane port | `8081`: admin API `/api/v1/*`, plus the web UI at `/` |
| Config file | `CALIBAN_CONFIG` (TOML), default `/etc/caliban/caliban.toml` |
| Admin auth (bootstrap) | `Authorization: Bearer $CALIBAN_ADMIN_TOKEN`. OIDC comes later. |
| Tenant API keys | `cal_<random>`, sent as `Authorization: Bearer cal_…` (OpenAI SDKs) or `x-api-key: cal_…` (Anthropic SDKs, `/v1/messages`). Only the SHA-256 hex hash is stored. |
| BYOK at rest | Envelope encryption, AES-256-GCM: tenant secrets (BYOK keys, datasource credentials) are sealed under a per-tenant DEK, and each DEK is wrapped by the KEK from `CALIBAN_KEK` (base64, 32 bytes). `CALIBAN_KEK_PREVIOUS` holds retired KEKs for opening only; `caliban keys rotate` re-wraps under the current KEK and `caliban keys status` shows which retired KEKs are still needed. Deleting a tenant destroys its DEK. HSM/KMS backends later |
| Postgres | `CALIBAN_DATABASE_URL` (control-plane store; unset = in-memory dev store). The config file seeds an empty database once; after that the database is the source of truth for tenants, keys, BYOK, providers, models, routes. Migrations run at startup. |
| Audit log | Every control-plane mutation appends a hash-chained `audit_log` row; `GET /api/v1/audit?limit=` (admin) |
| Split mode (CP) | `CALIBAN_SNAPSHOT_SIGNING_KEY` (base64 Ed25519 seed, `caliban gen-signing-key`) + `CALIBAN_ROUTER_TOKEN` enable `GET /api/v1/snapshot` (router token, ETag/304) |
| Split mode (router) | `caliban router --control-plane-url` / `CALIBAN_CONTROL_PLANE_URL`, `CALIBAN_ROUTER_TOKEN`, `CALIBAN_SNAPSHOT_PUBLIC_KEY` (comma-separated for rotation), `CALIBAN_SNAPSHOT_POLL_SECS` (default 10), `CALIBAN_SNAPSHOT_CACHE` (last good snapshot), `CALIBAN_ROUTER_ADDR` (default `0.0.0.0:8080`). Routers need the same `CALIBAN_KEK` as the CP. |
| Qdrant | `CALIBAN_QDRANT_URL` (default `http://qdrant:6334`) |
| Valkey | `CALIBAN_VALKEY_URL` (default `redis://valkey:6379`) |
| Logging | `CALIBAN_LOG` (tracing filter, e.g. `info,caliban=debug`) |
| Telemetry | OFF by default. OTLP (HTTP/protobuf) only if `OTEL_EXPORTER_OTLP_ENDPOINT` (or `OTEL_EXPORTER_OTLP_TRACES_ENDPOINT`) is set; `OTEL_SERVICE_NAME` (default `caliban`), `OTEL_EXPORTER_OTLP_HEADERS` and `OTEL_SDK_DISABLED=true` are honoured. Spans follow the OTel GenAI conventions (`chat {model}`, `gen_ai.*`, `caliban.*` attributes; child spans `route`, `pii`, `cache`, `upstream`). Prompt/response content is never recorded. An incoming W3C `traceparent` is continued. |
| Usage WAL | `CALIBAN_USAGE_WAL` (optional JSONL path; mount a volume, the image root is read-only) |
| Web console | `CALIBAN_WEB_DIR` (built `web/dist`; the console uses hash routing) |
| SDK env vars | `CALIBAN_API_KEY`, `CALIBAN_BASE_URL` (data plane), `CALIBAN_ADMIN_URL` + `CALIBAN_ADMIN_TOKEN` (control plane) |
| Health | `GET /healthz` (8080), `GET /api/v1/health` (8081); `caliban healthcheck` for shell-less images |
| PII NER model | `CALIBAN_PII_NER_DIR` (verified `pii_ner` artifact dir; binary built with `--features ner`), `CALIBAN_PII_NER_SESSIONS` (parallel ONNX sessions, default 1) |
| Outbound proxy | Standard `HTTPS_PROXY` / `NO_PROXY` are honoured for BYOK provider calls |

### Request extension (data plane)
Clients may add a `caliban` object to any OpenAI-compatible body. Clients that don't know about it ignore it.
```json
{ "model": "caliban/auto", "messages": [...],
  "caliban": { "pii": "reversible", "cache": "exact", "datasources": ["sales_dw"], "node": "invoice-triage", "max_cost_usd": 0.05, "zdr": true } }
```
Response headers: `x-caliban-request-id`, `x-caliban-routed-model`, `x-caliban-cache` (`hit|miss|bypass`), `x-caliban-pii-entities` (count), and `x-caliban-cost-usd` (non-streaming only); `/v1/messages` also sends `request-id`. All of them are CORS-exposed for browser clients.

### Anthropic Messages API (data plane)
`POST /v1/messages` and `POST /v1/messages/count_tokens` (approximate) run through the same pipeline as Chat Completions. Point an Anthropic SDK at `http://<host>:8080` with a `cal_…` key. If the routed model's provider is `kind = "anthropic"` the native body is passed through (`cache_control` kept, `anthropic-beta` forwarded, PII applied to text blocks); otherwise the request and response (including stream events) are translated. Errors use the Anthropic shape (`{"type":"error","error":{"type","message"}}`).

### Rate limits and token budgets
Configured in the TOML: `[limits]` holds defaults for every tenant, `[limits.tenants.<tenant_id>]` overrides individual fields. All fields are optional (unset = unlimited):

| Field | Meaning |
|---|---|
| `requests_per_minute` | GCRA request rate per tenant (bursts up to the per-minute value) |
| `key_requests_per_minute` | Same, per API key |
| `tokens_per_minute` | Token bucket refilling continuously over a minute |
| `tokens_per_day` | Tokens per UTC day |
| `usd_per_day` | Spend per UTC day (needs model prices) |

Before the upstream call Caliban **reserves** the prompt estimate (~4 bytes/token) plus `max_tokens` (1024 when unset) and the matching cost, and **settles** to the usage the upstream reports afterwards (streams included; if a client disconnects mid-stream, the output streamed so far is estimated). Exceeding a limit returns `429` with:
- `retry-after: <seconds>` and `x-caliban-ratelimit-scope: <field>` headers (CORS-exposed);
- OpenAI shape `{"error":{"type":"rate_limited","code":"<field>",…}}`, or Anthropic shape `{"type":"error","error":{"type":"rate_limit_error",…}}` on `/v1/messages`.

State is in memory per router process (exact for `standalone`). A shared Valkey store (`CALIBAN_VALKEY_URL`) for several routers is next; until then each router enforces its own counters. Quota store failures fail open.

### Known contract gaps (tracked)
- No `Idempotency-Key` support yet. SDKs must not retry non-idempotent POSTs on 502.
- Missing delete endpoints for tenants, API keys, datasources and nodes. No job-status endpoint for introspection. No node run endpoint (nodes are invoked via `caliban.node` on chat completions).
- Split mode (router and control plane as separate processes) works: routers poll the control plane's Ed25519-signed snapshot (`GET /api/v1/snapshot`), verify it, refuse rollbacks, and stay fail-static on errors (optionally restarting from `CALIBAN_SNAPSHOT_CACHE`). Gaps: polling only (no push/long-poll, so changes land within one poll interval); usage events stay on each router, so the control plane's `/api/v1/usage` only covers its own process; routers must run a version that understands the CP's config format (upgrade routers first). `standalone` remains the simplest on-prem shape.

### Virtual models
`caliban/auto` lets the router decide. A `<tenant-alias>` model name is resolved by tenant policy. Any other model ID must exist in the tenant's model registry.
