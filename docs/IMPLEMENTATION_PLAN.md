# CASI AI LABS — Free LLM Hub Implementation Plan

## Decision
Add a Free LLM API Hub to the CASI Dashboard. CASI issues customer-facing CASI API keys. These authenticate requests to the CASI gateway and are not provider credentials. Provider API keys are configured by authorized administrators and stored server-side in a secret store.

## Repository status
The connected repository was empty at integration start (only .gitkeep on main). This branch adds the architecture and provider seed, not production code. The Phase 03/04 application ZIP supplied in the conversation is not present in this GitHub repository. Integrating into that source requires importing/pushing the actual application first.

## Data source
Repository: https://github.com/casiailabsx-ux/Ll
README: https://github.com/casiailabsx-ux/Ll/blob/main/README.md
The README currently reports 537+ APIs from 31 providers and distinguishes permanent free tiers from renewable credits. Provider/model/limit data changes frequently and needs direct verification before enabling.

## Initial candidates
Groq, Google Gemini, Cerebras, Mistral, Hugging Face Router, NVIDIA NIM, Cloudflare Workers AI, Cohere, Z AI, SiliconFlow, OpenRouter and self-hosted Ollama.

## Security baseline
No provider secrets or generated keys committed. CASI keys are high-entropy, shown once, stored hashed, scoped, revocable, metered and rate-limited. Provider keys remain in secret manager/KMS. Egress allow-list and registry-controlled upstream URLs prevent SSRF.

## Next
Import the existing Phase 03/04 app into this repository or provide its actual repository URL/branch. Then implement against its real code, starting with API-key lifecycle and provider adapter interfaces.
