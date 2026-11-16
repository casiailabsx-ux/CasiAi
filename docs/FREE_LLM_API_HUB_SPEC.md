# CASI AI LABS — Free LLM API Hub & Customer API Keys

Status: architecture/specification branch; implementation not yet deployed.

## Product goal
Build a provider-aggregating LLM gateway. CASI issues its own API keys to authenticated CASI users. A CASI key authenticates requests to CASI; it is NOT a provider key. Provider credentials are separately configured by an authorized operator and remain server-side.

## Security/commercial distinction
- "Free" means a provider's published free tier or promotional quota, not unlimited usage.
- Never scrape, mint, resell, or proxy undocumented provider credentials.
- Users must respect provider terms, quotas, model availability and data-processing terms.
- No shared anonymous provider keys distributed to end users.
- Provider metadata must include source, last_verified_at, observed status and confidence.
- The source repository is a directory, not proof of entitlement to use a provider.

## Architecture
Client (Dashboard / SDK) -> FastAPI API Gateway -> CASI API Key Authentication -> RBAC/tenant/project scope -> quotas/rate limits -> request validation/model allow-list -> provider adapters -> upstream provider.
MongoDB can remain initially to fit the existing Phase 03/04 application. Redis should provide distributed limits and short-lived state. Provider credentials belong in a cloud secret manager/KMS, not ordinary application records.

## Provider catalogue seed
Candidates from casiailabsx-ux/Ll include Groq, Google Gemini, Cerebras, Mistral, Hugging Face Router, NVIDIA NIM, Cloudflare Workers AI, Cohere, Z AI, SiliconFlow, OpenRouter and self-hosted Ollama.
The source README reports 537+ APIs across 31 providers and separates permanent free tiers from renewable credits. Re-verify each provider directly before enabling it. Model IDs, rate limits and URLs change; do not hard-code published numbers as contractual user quotas.

## CASI API key lifecycle
- Prefix: casi_; generate at least 32 cryptographically random bytes.
- Display raw key exactly once at creation; never provide retrieval afterwards.
- Store only a keyed HMAC-SHA-256 digest (server-side pepper) or equivalent strong one-way verifier.
- Persist key_id, owner_user_id, tenant_id, project_id, prefix_hint, scopes, status, created_at, expires_at, last_used_at, revoked_at, rotation_parent_id, allowed_models and budget policy.
- Constant-time digest comparison.
- Support create, list masked, revoke, rotate and expire.
- Scopes: models:read, chat:completions, usage:read.
- Bind key to authenticated owner, tenant and project; never trust tenant_id from request body.
- Require verified account and plan entitlement for key issuance.
- Optional IP allow-list and key expiry.

## Public API
Base URL: https://api.casi-ai-labs.com/v1
Dashboard session endpoints:
- POST /api-keys
- GET /api-keys
- POST /api-keys/{key_id}/revoke
- POST /api-keys/{key_id}/rotate
Inference endpoints authenticated with Authorization: Bearer casi_...:
- GET /models
- POST /chat/completions
- GET /usage
Use OpenAI-compatible request/response shapes where possible. SSE streaming requires cancellation on client disconnect, bounded buffers and normalized errors.

## Provider adapter contract
Each adapter implements provider_id, capabilities, list_models(), health_check(), create_chat_completion() and normalize_error(). Add provider-specific request/response transformation for non-OpenAI-compatible APIs. Do not assume all entries share one protocol.

## Dashboard UX
Models & APIs -> Free LLM API Hub:
1. Provider directory with free-tier classification, modalities, model count, verification timestamp, provider signup link and health.
2. Model explorer with filters for provider, capabilities, context window and tier.
3. Operator-only provider connection form. Test credentials before activation; secrets masked after save.
4. My CASI API Keys: create key with project/scopes/model allow-list/expiry/budget; display once; masked list; last used; usage; revoke and rotate.
5. Playground with enabled provider/model selection and test prompt.
6. Usage: requests, tokens, quotas, rate-limit events and estimated costs where applicable.
7. Real health checks with timestamps. Never fake live metrics.

## Provider credential handling
Provider keys are operator-managed secrets, separate from customer CASI keys. Use a secret manager/KMS; fallback only to envelope encryption with a KMS-managed key. Never log Authorization headers, provider keys, prompts containing secrets or raw keys. Mask secrets. Validate TLS. Use registry-controlled upstream URLs and egress allow-lists to prevent SSRF. Never accept arbitrary user-supplied base URLs in the shared gateway.

## Quotas and abuse controls
- Per-CASI-key RPM/TPM; per-user/tenant daily and monthly budgets.
- Per-model concurrency, maximum request/body/token size and timeout.
- Track provider quota separately from CASI user quota.
- Redis-backed distributed counters and durable usage ledger.
- Circuit breaker, bounded retry with jitter, backoff for 429/5xx.
- Admin kill switch per provider/model/key.
- Enforce entitlement and budget before dispatch.
- Do not promise unlimited free access; upstream provider quota is provider-controlled.

## Logical data model
Provider: provider_id, name, adapter_type, base_url, signup_url, free_tier_class, terms_url, capabilities, enabled, health_state, last_verified_at, catalog_source.
Model: model_id, provider_id, display_name, context_window, modalities, capabilities, lifecycle_state, free_tier_class, limits_metadata, last_verified_at.
ProviderSecretRef: provider_id, secret_ref, credential_status, created_at, rotated_at. No plaintext credential in DB.
CasiApiKey: key_id, key_hash, key_prefix, owner_user_id, tenant_id, project_id, scopes, allowed_models, status, created_at, expires_at, last_used_at, revoked_at, budget_policy.
UsageEvent: request_id, key_id, owner_user_id, tenant_id, provider_id, model_id, input_tokens, output_tokens, latency_ms, status_code, estimated_cost, created_at.

## Security requirements
Address OWASP API Security: object-level authorization, broken auth, resource consumption, SSRF, misconfiguration, inventory management and unsafe upstream API consumption.
Require session auth for key-management, tenant isolation, hashed CASI keys, KMS-backed upstream secrets, rate/budget controls, egress allow-list, strict model registry, CORS allow-list, CSRF for cookie mutations, TLS, CSP/security headers, dependency/secret scans, audit create/revoke/rotate and secret redaction.

## Phased implementation
A: Provider catalogue and contracts (no live secrets).
B: CASI API key lifecycle + tenant-aware RBAC.
C: Gateway + adapters; start with one verified provider (e.g. Groq) and then Gemini native adapter.
D: Redis quotas, usage metering and audit.
E: Dashboard UX, playground and model comparison.
F: More providers, embeddings/RAG and production hardening.
Every phase requires build, unit/integration tests, security review, docs and verified backup. Stop for owner approval after each phase.

## Acceptance criteria
- CASI key shown once; raw key never stored.
- Revoked/expired keys fail immediately.
- Scope/project/tenant boundaries enforced server-side.
- Provider secrets never reach browser or customer.
- Distributed throttling and usage metering work across instances.
- Upstream errors normalized without credential leakage.
- Dashboard distinguishes free tier, promotional credits, paid and self-hosted.
- Catalogue includes sources and verification dates.
- Test key lifecycle, auth bypass, quota concurrency, adapter normalization, SSE disconnect and secret redaction.

## Non-goals
No automatic generation of provider-owned keys, no bypassing signup/quota/verification/terms, no shared public upstream credential, no client-side provider secrets, and no enabling providers before real credential-backed tests.
