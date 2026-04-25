# Glossary

Living doc. Add new terms here *first* when you encounter them; promote to a full topic note if the term grows a life of its own.

## Core LLM mechanics

- **Token** — the atomic unit an LLM reads and emits. Roughly ¾ of a word on average for English. Pricing and context windows are measured in tokens.
- **Context window / context length** — max tokens the model can attend to in one request. Covers input + output combined (usually). Large windows = more memory + slower.
- **Embedding** — a dense vector (e.g., 1024 floats) representing the "meaning" of a chunk of text. Similar meanings → nearby vectors. Enables RAG.
- **KV cache** — cached key/value tensors from prior tokens so the model doesn't recompute them for each new token. Dominates GPU memory use at long context lengths.
- **Quantization** — shrinking model weights from FP16 to INT8/INT4 etc. Trades quality for memory/speed. Common formats: AWQ, GPTQ, FP8, GGUF (Q4_K_M = 4-bit, "medium" variant). Deep dive: [gguf-quantization.md](./gguf-quantization.md).
- **GGUF** — single-file model format used by llama.cpp. Contains quantized weights + tokenizer + metadata. Replaced older GGML. Runs on CPU (and GPU via CUDA/Metal). The format you pull off HuggingFace for llama.cpp-style serving. Deep dive: [gguf-quantization.md](./gguf-quantization.md).
- **OpenAI-compatible API** — the de-facto REST interface for LLMs: `/v1/models`, `/v1/chat/completions`, `/v1/embeddings`, SSE streaming via `stream:true`. llama.cpp, vLLM, TGI, Ollama, LiteLLM all speak it — so clients are portable across backends. This swap-ability is the whole premise of a model gateway (Phase 3).

## Serving & latency

- **TTFT (time-to-first-token)** — wall-clock from request sent → first token streamed back. The user-perceived latency.
- **TPOT (time-per-output-token)** — steady-state per-token latency after streaming starts. Determines total response time.
- **Continuous batching** — in-flight requests can join an in-progress batch mid-generation. vLLM's killer feature. Contrast: static batching (wait for N requests, run them together).
- **PagedAttention** — paging the KV cache like OS virtual memory. Lets vLLM allocate KV in non-contiguous blocks, driving much higher GPU utilization.
- **Speculative decoding** — use a small "draft" model to predict several tokens ahead; big model verifies in one forward pass. Throughput boost if draft is accurate.
- **Scale-to-zero** — pod count drops to 0 after idle. Next request triggers cold start. Critical for $1/hr GPUs.
- **Cold start** — time from "request arrives at zero-pod service" → "first token returned". Dominated by model load (10–60s+ for large models).

## RAG

- **RAG (Retrieval-Augmented Generation)** — fetch relevant docs at query time and stuff them into the prompt. Alternative to fine-tuning for domain knowledge.
- **Chunk** — a slice of a source doc (e.g., 500 tokens) that gets embedded and indexed as one unit.
- **Vector DB** — stores embeddings and supports nearest-neighbor search. Options: pgvector, Qdrant, Weaviate, Pinecone.
- **Reranker** — a second-stage model that re-orders retrieved chunks by relevance. Improves quality at latency cost.
- **Hybrid search** — combining vector search with keyword (BM25) search. Usually wins over either alone.

## Safety & quality

- **Guardrail** — input or output filter. Input guardrails block prompt injection / unsafe asks. Output guardrails redact PII / block toxic content.
- **Prompt injection** — user input (or content in a doc the LLM reads) contains instructions that hijack the model. Direct, indirect (via RAG docs), or multi-turn.
- **PII (Personally Identifiable Information)** — emails, phones, SSNs, credit cards, names. Must often be scrubbed from inputs AND outputs.
- **Eval** — automated quality test on a prompt. Scoring methods: exact match, regex, LLM-as-judge, rubric, semantic similarity.
- **LLM-as-judge** — using a stronger LLM to grade a weaker one's outputs. Cheap eval at scale; has its own biases (verbosity, self-preference).
- **Ragas** — eval library specific to RAG: context recall, faithfulness, answer relevance.

## Platform / infra

- **IDP (Internal Developer Platform)** — self-service portal (e.g., Backstage) exposing golden paths to product teams. "Onboard my team" as a form, not a ticket.
- **Golden path** — the paved road for a common task. Opinionated, batteries-included. Reduces decisions product teams have to make.
- **Gateway** — single HTTP front door for all model calls. Handles routing, fallback, rate limits, spend tracking, caching. Examples: LiteLLM, Portkey.
- **OTel GenAI semantic conventions** — OpenTelemetry standard attributes for LLM calls: `gen_ai.request.model`, `gen_ai.usage.input_tokens`, etc.
- **ADR (Architecture Decision Record)** — short doc capturing context + decision + trade-offs for a specific choice. Immutable once accepted.
