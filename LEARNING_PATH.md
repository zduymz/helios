# Learning Path — 10 phases

Each phase ends with a **working, committed artifact** and **updated notes**. No phase ends with "I read about X."

## Skill → Phase → Note-file cheat sheet

| Phase | Skill | Primary note | Decision record(s) | Estimate |
|---|---|---|---|---|
| 0 | Setup | `00-glossary.md` | — | 0.5 day |
| 1 | Inference serving | `01-vllm.md`, `02-kserve.md` | 0002 | 1 week |
| 2 | GPU scheduling | `03-gpu-scheduling.md` | 0003 | 3–4 days |
| 3 | Model gateway | `04-litellm.md` | 0004 | 3 days |
| 4 | Observability | `05-otel-genai.md` | 0005 | 4–5 days |
| 5 | RAG | `06-rag.md` | 0001, 0006 | 1–1.5 weeks |
| 6 | Guardrails | `07-guardrails.md` | — | 3–4 days |
| 7 | Evals | `08-evals.md` | 0007 | 3–4 days |
| 8 | IDP | `09-backstage.md`, `00-platform-thesis.md` | 0008 | 1 week |
| 9 | Portfolio | README deep-dive | — | 2 days |

Total realistic calendar: **2–3 months** working evenings/weekends.

---

## Phase 0 — Setup (½ day)

**Goal**: repo + local cluster + empty second brain scaffolded.

**Build**:
- `kind` (or `k3d`) cluster locally; no GPU yet, mock inference first.
- ArgoCD installed, pointed at `infra/argocd/`.
- Empty `notes/` structure with `00-glossary.md` pre-seeded.

**Notes to capture**:
- `kind` config, including the GPU passthrough settings you'll need in Phase 1d.
- ArgoCD app-of-apps pattern — 10 lines explaining it.

**Done when**: `kubectl get ns` shows `argocd` + placeholder `helios` namespace. Repo pushed to GitHub.

---

## Phase 1 — Inference serving (1 week)

**Skill**: vLLM + KServe. The foundation.

**See [PHASE_1.md](./PHASE_1.md) for the full step-by-step walkthrough.**

**Build**:
1. Day 1–2: llama.cpp server (Docker, CPU) with a tiny model → understand the OpenAI-compatible API.
2. Day 3: same server, now as a kind `Deployment`. Measure TTFT.
3. Day 4–5: install KServe. Wrap the server as an `InferenceService`. Scale-to-zero.
4. Day 6–7: rent a GPU node (~$1/hr for 2 hrs) → Llama-3.1-8B with **vLLM** on KServe. Benchmark. Shut down.

**Aha moments**:
- Throughput vs latency: batching improves throughput, hurts TTFT.
- KV cache is why LLM serving is weird — most GPU memory is cache, not weights.
- The OpenAI API is the de-facto interface; implementations are swappable.

**Done when**: one `kubectl apply -f` deploys a working LLM endpoint that scales to zero.

---

## Phase 2 — GPU scheduling (3–4 days)

**Skill**: make GPUs shareable across teams without fistfights.

**Build**:
1. NVIDIA GPU Operator on the GPU node.
2. Enable time-slicing (simpler) or MIG (if supported) — split one A10/A100 into multiple logical GPUs.
3. Kueue with 2 `ClusterQueues` (team-a, team-b), fair-sharing.
4. Submit competing jobs from both teams; observe pre-emption and queuing.

**Notes to capture**:
- `03-gpu-scheduling.md`: MIG profiles, time-slicing config, `nvidia.com/gpu` semantics, Kueue's `ResourceFlavor` + `ClusterQueue` + `LocalQueue` mental model.
- Spot GPU strategy: tolerations, `PodDisruptionBudget`, checkpointing.
- `decisions/0003-time-slicing-vs-mig.md`.

**Done when**: two teams submit 5 jobs each; Kueue serves them fairly; no OOMs.

---

## Phase 3 — Model gateway (3 days)

**Skill**: LiteLLM — the single front door for all models.

**Build**:
1. Deploy LiteLLM proxy on the cluster.
2. Backends: `local-llama` (Phase 1), `anthropic/claude-opus-4-7`, `openai/gpt-4o-mini`.
3. Per-team virtual keys with budgets, rate limits, fallback chains.
4. Test failover: kill vLLM → auto-route to Claude. Test budget exhaustion.
5. Turn on semantic caching (Redis-backed). Measure cache hit rate.

**Notes to capture**:
- `04-litellm.md`: config schema, router strategies, fallback chains, virtual keys, spend tracking.
- `decisions/0004-litellm-vs-portkey.md`.
- Table: **each LiteLLM feature → the product-team pain point it solves**. This is the document you show your manager to justify the platform.

**Done when**: `curl litellm/v1/chat/completions -H "Authorization: Bearer sk-team-a-..."` works; spending appears per-team in LiteLLM admin UI.

---

## Phase 4 — Observability (4–5 days)

**Skill**: OpenTelemetry GenAI + Langfuse. See every request, cost, failure.

**Build**:
1. OpenTelemetry Collector.
2. Langfuse (self-hosted, Postgres + ClickHouse).
3. Wire LiteLLM → OTel → Langfuse using the **OTel GenAI semantic conventions** (`gen_ai.request.model`, `gen_ai.usage.input_tokens`, …).
4. Tag every request with `team`, `feature`, `user_id`.
5. Three Grafana panels: $ by team (7d), p50/p95/p99 TTFT by model, error rate by team+model.
6. Budget alert: any team >$50/day → Slack webhook.

**Notes to capture**:
- `05-otel-genai.md`: the `gen_ai.*` semantic conventions, span hierarchy for agent calls, why traces not just metrics.
- `decisions/0005-langfuse-over-datadog-llm.md`.
- `incidents/YYYY-MM-DD-first-cost-spike.md`.

**Done when**: every gateway call appears in Langfuse with cost + latency + team tags; you can answer "which team spent most yesterday?" in 5 seconds.

---

## Phase 5 — RAG infra (1–1.5 weeks)

**Skill**: pgvector + embedding pipeline + retrieval API.

**Build**:
1. Postgres with pgvector (via CloudNativePG operator).
2. Schema with per-row `team_id` and RLS policies.
3. HNSW index on `VECTOR(1024)` column.
4. Embedding pipeline as an Argo Workflow: S3 → chunk → embed → upsert. Idempotent on content hash.
5. `/retrieve` HTTP service with team-scoped RLS enforcement.
6. Wire `apps/demo-support-bot` to consume gateway + retrieve.
7. RAG-specific evals with Ragas (context recall, faithfulness, answer relevance).

**Aha moments**:
- 80% of RAG quality problems are chunking/retrieval, not the LLM.
- Multi-tenant ACLs are the hidden hard part. Pick RLS or pre-filtered search — one of them.

**Done when**: demo-support-bot answers with citations; team-a cannot retrieve team-b's chunks.

---

## Phase 6 — Guardrails (3–4 days)

**Skill**: input + output safety filters in the gateway.

**Build**:
1. Llama Guard 3 as a small inference service.
2. Microsoft Presidio for PII detection/redaction (CPU).
3. LiteLLM pre/post hooks: Presidio → Llama Guard (input); Presidio → Llama Guard (output).
4. 10 adversarial test inputs (injection, jailbreak, PII leak). Verify blocks.
5. Opt-out per team for internal use cases.

**Notes to capture**:
- `07-guardrails.md`: injection taxonomy (direct, indirect-via-RAG, multi-turn), why defense in depth beats any single filter, the 100–300ms latency tax.
- `incidents/YYYY-MM-DD-first-injection-bypass.md`.

**Done when**: adversarial suite 90%+ block rate; p95 latency added by guardrails <300ms.

---

## Phase 7 — Evals as CI gates (3–4 days)

**Skill**: Promptfoo blocking PRs on quality regressions.

**Build**:
1. 30-case golden dataset for `demo-support-bot`.
2. `evals/promptfoo.yaml` with exact-match, LLM-as-judge rubric, regex asserts.
3. GitHub Actions workflow on PRs touching `apps/` or `platform/gateway/`.
4. Gate: overall score >= 0.85.
5. Intentionally regress a prompt → watch CI block → fix → green.
6. Pipe Promptfoo output → Langfuse for eval score trendlines.

**Notes to capture**:
- `08-evals.md`: scoring methods, dataset building (synthetic vs production-derived), the "LLM judging LLM" trap, eval drift.
- `decisions/0007-llm-as-judge-model-choice.md` — always judge with a stronger model.

**Done when**: a PR with a worse prompt gets auto-blocked with a readable eval diff.

---

## Phase 8 — Platform layer / IDP (1 week)

**Skill**: Backstage + golden path templates. The whole point of everything before.

**Build**:
1. Deploy Backstage.
2. Template: *"Onboard my team to the AI platform"*. Inputs: team name, monthly budget, models needed. Output: one PR creating LiteLLM virtual key, Langfuse project, pgvector schema, ArgoCD app, starter `promptfoo.yaml`.
3. Template: *"Add RAG to my feature"*. Input: S3 path, team. Output: Argo Workflow embedding pipeline PR.
4. Tech Insights scorecard: spend, eval-score trend, guardrail block rate, incident count.

**Notes to capture**:
- `09-backstage.md`: catalog-info.yaml, scaffolder actions, plugins, entity model.
- `decisions/0008-backstage-vs-port-vs-custom.md`.
- **Headline note**: `00-platform-thesis.md` — 300 words on what "platform engineering" means to you now. This is the interview document.

**Done when**: a new team can onboard and ship a RAG endpoint in ONE PR, with cost limits + observability + guardrails + evals all wired automatically.

---

## Phase 9 — Demo + portfolio narrative (2 days)

1. 5-minute Loom: new team onboards to Helios and ships a RAG bot, live.
2. Deep root `README.md`: architecture, decisions (link ADRs), metrics (p95 TTFT, per-team cost, eval score).
3. Blog/LinkedIn per major phase — each derived from the `notes/` that already exist.

---

## How to sustain this over 2–3 months

- **Daily log, 15 minutes, non-negotiable**: `notes/log/YYYY-MM-DD.md`. Three bullets.
- **One ADR per real decision** (>30 min research).
- **An incident file for every "oh" moment**, however small.
- **Cap cloud GPU spend**: AWS budget alarm at $50/mo before Phase 1d.
- **Don't skip phases**. Phase 2 is where senior platform people are made.
