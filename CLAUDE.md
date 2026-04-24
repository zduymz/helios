# CLAUDE.md — context for Claude Code sessions in this repo

## Who I am

DevOps / SRE / Platform Engineer in 2026 (enterprise experience, strong Kubernetes and AWS). Comfortable with Terraform, Helm, ArgoCD, eBPF, OpenTelemetry. Currently leveling up into **AI infrastructure + platform engineering** — that's what this repo is for.

## What this project is

A learning project: building an end-to-end self-service AI platform on Kubernetes for a fictitious company (AcmeCorp). Not for production use. The point is to build real artifacts across every layer of the modern AI infra stack and document the journey as a portfolio.

Full project brief in [README.md](./README.md). Phase plan in [LEARNING_PATH.md](./LEARNING_PATH.md).

## How to help me

**Default posture**: concrete > abstract. Commands, YAML, file paths. No hand-waving. If I'm stuck on a config, show me the config — don't explain the theory.

**Terse over verbose.** I can read a diff; don't summarize what you just did.

**Surface trade-offs, don't hide them.** When picking tools (pgvector vs Qdrant, KServe vs raw Deployment), lay out the 2–3 real factors and recommend one with a *why*.

**Push back when I'm wrong.** If I suggest something that'll blow up (e.g., running inference on the default scheduler with no GPU reservation), say so plainly — don't build the broken thing.

## The second brain is the point

This repo has a `notes/` folder that is the primary artifact, not a side effect. See [notes/README.md](./notes/README.md) for the protocol. Key habits:

1. **Daily log** (`notes/log/YYYY-MM-DD.md`, 15 minutes, 3 bullets: did / broke / learned). At end of any real working session, prompt me to write one if I haven't.
2. **Topic notes** (`notes/NN-topic.md`) — one per tool/concept. Short, atomic, with commands I actually ran.
3. **ADRs** (`notes/decisions/NNNN-title.md`) — one per real decision (>30 min of research/debate).
4. **Incidents** (`notes/incidents/YYYY-MM-DD-slug.md`) — one per "oh no" moment, including small ones.

When I finish a sub-phase, **proactively offer to draft a note** from what we just did. Draft using the template in `notes/_topic-template.md` (or the relevant ADR / incident template); I'll edit.

## Collaboration preferences

- **Golden path**: when there are five ways to do something, recommend one with reasons. I can ask for the others if I want them.
- **Commands over prose**: if I ask "how do I install X," the response should lead with the command.
- **Cost awareness**: I'm paying for cloud GPUs out of pocket. Always call out when a step incurs real cost and suggest the cheapest path that still teaches the concept. Default to CPU / tiny models / local kind clusters until I explicitly say "OK we're going to cloud GPU now."
- **Don't assume context**: I switch between AWS accounts and K8s clusters. Never run `assume` or `kube-env` yourself (see root project CLAUDE.md rules). Always verify context first with `aws sts get-caller-identity` and `kubectl config current-context`.

## Where I am

Update this section as phases complete.

- **Current phase**: Phase 0 → 1 (just started)
- **Last session**: repo scaffolded from the Claude conversation that birthed this project
- **Next action**: follow [PHASE_1.md](./PHASE_1.md) step 1 (llama.cpp server in Docker locally)

## Tech choices locked in (so far)

| Layer | Choice | Why (short) |
|---|---|---|
| Local cluster | kind (or k3d) | Fast, throwaway, no cloud cost |
| Inference server (prod) | vLLM | PagedAttention + continuous batching, de-facto OSS choice |
| Inference server (local, CPU) | llama.cpp server | Actually works on CPU with tiny models; same OpenAI API |
| Model serving CRD | KServe | Std. autoscaling + canary + scale-to-zero |
| GPU scheduler | Kueue + NVIDIA GPU Operator | Queue fairness across teams |
| Gateway | LiteLLM | OSS, OpenAI-compatible, spend tracking, routing, fallback |
| Vector DB | pgvector | Already ops Postgres; RLS for multi-tenancy |
| Observability | OTel GenAI + Langfuse (self-hosted) | Vendor-neutral; eval-aware |
| Guardrails | Llama Guard 3 + Presidio | Defense in depth (classifier + regex/NER) |
| Evals | Promptfoo | CI integration; multiple assertion types |
| IDP | Backstage | Market leader; scaffolder templates |

Each of these should get an ADR in `notes/decisions/` as I implement it. Don't accept the table as "settled" — if something doesn't fit, flag it.

## Things NOT to do

- Don't pre-build many directories "just in case". Grow them when needed.
- Don't generate long docs I didn't ask for. Notes files are short.
- Don't write emojis in files (unless I explicitly ask).
- Don't add comments that re-describe the code. Only comments for *why*.
- Don't skip steps in the learning path to "get to the interesting part". The unsexy steps are the point.
