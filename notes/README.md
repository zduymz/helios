# notes/ — the second brain

This folder is the **primary artifact** of the Helios project, not a side effect. The code will be rewritten and thrown away. These notes are what you keep.

## Four types of notes

| Type | Location | When to write |
|---|---|---|
| **Topic note** | `NN-topic.md` | One per tool/concept. Atomic, short, with commands you actually ran. |
| **ADR** | `decisions/NNNN-slug.md` | Any decision that took >30 min of research/debate. |
| **Incident** | `incidents/YYYY-MM-DD-slug.md` | Every "oh no" moment, however small. Your own mistakes are the best teacher. |
| **Daily log** | `log/YYYY-MM-DD.md` | 15 minutes at end of each working session. Three bullets: did / broke / learned. |

## Rules

1. **If you Googled something and it took >10 minutes, it goes in notes.** Future-you will thank present-you.
2. **Keep topic notes short.** 1 page max. Lots of small notes > a few big ones.
3. **Commands go in notes, not screenshots.** Copy-pasteable > pretty.
4. **Always include *why*, not just *what*.** What-is is in the docs; your note's value is the context.
5. **Update, don't append forever.** If you learn more about vLLM, edit `01-vllm.md`. Don't start `01-vllm-v2.md`.
6. **ADRs are immutable.** Once accepted, don't retroactively edit the reasoning. Write a new ADR that supersedes it.

## Templates

- [`_topic-template.md`](./_topic-template.md)
- [`decisions/_template.md`](./decisions/_template.md)
- [`incidents/_template.md`](./incidents/_template.md)
- [`log/_template.md`](./log/_template.md)

## Index

### Topic notes
- [00 — Glossary](./00-glossary.md)
- [01 — Inference serving (llama.cpp, Phase 1a scaffold — renames to `01-vllm.md` after Phase 1d)](./01-inference-serving.md)
  - [GGUF & quantization](./gguf-quantization.md) — reference, cross-cuts phases
- [02 — KServe (`InferenceService` on Knative)](./02-kserve.md)
- `03-gpu-scheduling.md` (Phase 2)
- `04-litellm.md` (Phase 3)
- `05-otel-genai.md` (Phase 4)
- `06-rag.md` (Phase 5)
- `07-guardrails.md` (Phase 6)
- `08-evals.md` (Phase 7)
- `09-backstage.md` (Phase 8)
- `00-platform-thesis.md` (Phase 8, headline)

### ADRs
- [0001 — KServe `InferenceService` (Serverless) over raw `Deployment`](./decisions/0001-kserve-over-raw-deployment.md)

### Incidents
(empty — add as they happen)

### Daily log
- [2026-04-24](./log/2026-04-24.md) — Phase 1a (llama.cpp in Docker)
- [2026-04-25](./log/2026-04-25.md) — Phase 1b + 1c (kind Deployment → KServe `InferenceService`)
