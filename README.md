# Helios — a self-service AI platform on Kubernetes

A personal learning project: build a realistic AI platform for a fictitious company (**AcmeCorp**) with 12 product teams all wanting to ship LLM features. You are the 1-person AI platform team. Your job: make it easy, cheap, and safe for any team to go from *"I want RAG on my feature"* to *"shipped in prod"* via **one PR** — with cost, safety, and quality guardrails baked in.

By the end you will have built — and be able to speak credibly about — every layer of the modern AI infrastructure stack.

## Why this project

For a DevOps / SRE / Platform Engineer in 2026, the highest-leverage underserved skill set is **AI infra + platform engineering together**: running LLM workloads on Kubernetes exposed via a self-service internal developer platform. That intersection is exactly what this project builds.

## Architecture (final state)

```
  Developer
     │
     ▼
 Backstage IDP  ──►  PR template (Crossplane/Score)  ──►  ArgoCD  ──┐
                                                                    │
                                                                    ▼
                                                             Kubernetes cluster
                                                             ┌──────────────────────────────┐
 Product app ─────► LiteLLM Gateway ─┬──► vLLM + KServe (self-hosted Llama-3.1-8B on GPU)
                    │                │
                    │                └──► Anthropic / OpenAI / Bedrock (external)
                    │
                    ├──► Llama Guard + Presidio (guardrails: PII + injection)
                    │
                    ├──► pgvector (RAG store)   ◄──  Argo Workflow: embedding pipeline
                    │
                    └──► OpenTelemetry ──► Langfuse (observability, evals, cost attribution)

 CI (GitHub Actions) ──► Promptfoo ──► block PR if eval scores regress
```

## Repo layout

```
helios/
├── README.md                     ← this file
├── CLAUDE.md                     ← context for Claude Code sessions
├── LEARNING_PATH.md              ← the 10-phase roadmap
├── PHASE_1.md                    ← current phase, in detail
├── infra/
│   ├── local/                    ← kind/k3d manifests for dev
│   ├── aws/                      ← terraform for EKS + GPU node group (later)
│   └── argocd/                   ← ArgoCD app-of-apps
├── platform/
│   ├── gateway/                  ← LiteLLM config + helm values
│   ├── serving/                  ← KServe InferenceServices, vLLM configs
│   ├── rag/                      ← pgvector, embedding workflows
│   ├── guardrails/               ← Llama Guard + Presidio + policies
│   ├── observability/            ← OTel collector, Langfuse, dashboards
│   └── idp/                      ← Backstage templates (golden paths)
├── apps/
│   └── demo-support-bot/         ← "fake product team" that consumes the platform
├── evals/
│   ├── datasets/
│   └── promptfoo.yaml
└── notes/                        ← ★ SECOND BRAIN ★
    ├── README.md                 ← the note-taking protocol
    ├── 00-glossary.md
    ├── 01-vllm.md
    ├── ...
    ├── decisions/                ← ADRs
    ├── incidents/                ← your own mistakes
    └── log/                      ← 15-min daily "what I learned"
```

## Status

| Phase | Skill | Status |
|---|---|---|
| 0 | Setup (repo, local cluster, glossary) | ☐ |
| 1 | Inference serving (vLLM + KServe) | ☐ ← **start here** |
| 2 | GPU scheduling (GPU operator, Kueue) | ☐ |
| 3 | Model gateway (LiteLLM) | ☐ |
| 4 | Observability (OTel + Langfuse) | ☐ |
| 5 | RAG (pgvector + embedding pipeline) | ☐ |
| 6 | Guardrails (Llama Guard + Presidio) | ☐ |
| 7 | Evals as CI gates (Promptfoo) | ☐ |
| 8 | Platform layer / IDP (Backstage) | ☐ |
| 9 | Demo + portfolio writeup | ☐ |

See **[LEARNING_PATH.md](./LEARNING_PATH.md)** for the full roadmap and **[PHASE_1.md](./PHASE_1.md)** for today's detailed instructions.

## Ground rules

- **Cost discipline**: default to local CPU models (tiny). Rent cloud GPUs only for specific benchmarking sessions; shut them down immediately. Set an AWS budget alarm at $50/mo before Phase 1d.
- **Second brain first**: if you Googled something and it took >10 min, it goes in `notes/`. See [notes/README.md](./notes/README.md) for the protocol.
- **Don't skip phases**. Phase 2 (GPU scheduling) is unsexy but it's where senior platform people are separated from junior ones.
- **One ADR per real decision**. Future-you will redo the research otherwise.
