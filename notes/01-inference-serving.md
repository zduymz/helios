# 01 — Inference serving (llama.cpp server on CPU)

> Scaffold note for Phase 1a. Will be **renamed to `01-vllm.md`** after Phase 1d when vLLM replaces llama.cpp as the production backend. The cross-cutting idea below (OpenAI API as lingua franca) survives the rename.

## What it is (1 sentence)
A local HTTP server that loads a GGUF-quantized LLM and exposes it behind the OpenAI-compatible `/v1/*` API, so any OpenAI-style client can talk to it unchanged.

## Why it exists (the problem it solves)
Running vLLM on a laptop CPU is painful (build-from-source, slow). `llama.cpp` was built for exactly this: CPU-first, quantized, fast to start. It gives you a real LLM endpoint to iterate against without burning GPU dollars, and speaks the same API surface vLLM/TGI/Ollama/LiteLLM do — so code written against it ports cleanly when we swap backends.

## The headline insight: OpenAI API as lingua franca
- **The interface is stable; the implementation is swappable.** llama.cpp today → vLLM on a GPU tomorrow → Anthropic/OpenAI via LiteLLM after that. Same client code.
- This is why Phase 3 (LiteLLM gateway) works at all: every backend just speaks `/v1/chat/completions`, so the gateway is a thin router + policy layer, not a translation layer.
- Corollary: resist temptation to write to a proprietary inference API. Always prefer the OpenAI shape, even when serving your own model.

## Key concepts
- **GGUF** — llama.cpp's single-file packaging (weights + tokenizer + metadata). Q4_K_M is the go-to "4-bit, reasonable quality" quant. Deep dive: [gguf-quantization.md](./gguf-quantization.md).
- **`-c` / context length** — KV cache size cap. Bigger = more RAM, longer conversations supported.
- **SSE streaming** — `"stream": true` flips the response to `text/event-stream`; each `data: {...}` line is one token chunk. TTFT = time to the first such chunk.
- **Chat template** — baked per-model (ChatML, Llama-3, Qwen, …). Wrong template → nonsense output. llama.cpp usually picks it from GGUF metadata; override with `--chat-template` if needed.

## Commands I actually ran (Phase 1a)
```bash
# 1. Pull a tiny model (~400 MB)
mkdir -p ~/helios-models && cd ~/helios-models
curl -L -o qwen2.5-0.5b-instruct-q4_k_m.gguf \
  https://huggingface.co/bartowski/Qwen2.5-0.5B-Instruct-GGUF/resolve/main/Qwen2.5-0.5B-Instruct-Q4_K_M.gguf

# 2. Run the server
docker run -d --name helios-llm \
  -p 8080:8080 \
  -v ~/helios-models:/models \
  ghcr.io/ggml-org/llama.cpp:server \
  -m /models/qwen2.5-0.5b-instruct-q4_k_m.gguf \
  --host 0.0.0.0 --port 8080 \
  -c 4096

# 3. Sanity-check
curl -s localhost:8080/v1/models | jq
curl -s localhost:8080/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"qwen2.5-0.5b","messages":[{"role":"user","content":"In one sentence, what is KV cache?"}]}' \
  | jq '.choices[0].message.content'

# 4. Measure TTFT (time_starttransfer ≈ first-byte time for SSE)
curl -sN -o /dev/null \
  -w 'TTFT ~= %{time_starttransfer}s  total=%{time_total}s\n' \
  localhost:8080/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"qwen2.5-0.5b","stream":true,
       "messages":[{"role":"user","content":"Count from 1 to 20."}]}'
```

## Numbers from my run (Qwen2.5-0.5B-Q4_K_M, laptop CPU)
- TTFT: ~15 ms
- Total (count to 20): ~605 ms

Reference points for later comparison: Phase 1b will run the same image in kind (expect same numbers + container overhead). Phase 1d runs Llama-3.1-8B on an A10 — TTFT should stay similar for small prompts, but throughput under concurrency will be in a different league thanks to PagedAttention + continuous batching.

## Phase 1b — same server, in kind as a Deployment

Manifest: [../platform/serving/llama-cpp-deployment.yaml](../platform/serving/llama-cpp-deployment.yaml). Namespace `helios`, NodePort 30080, model mounted from the kind node's `/models` via `hostPath`.

Numbers (1× control-plane kind cluster, image and model already cached on the node):
- `kubectl rollout status` deploy → Ready: **~12 s**
- Pod delete → replacement pod Ready: **~12 s**

Both endpoints work: `localhost:30080/v1/models` (NodePort) and `kubectl port-forward svc/llama-cpp 8088:8080`.

### Why `hostPath` is dev-only (and what to do in prod)
`hostPath` ties the pod to one specific node — a 5 GB GGUF baked into one node doesn't migrate, doesn't scale, doesn't survive a node replacement. Two production-grade alternatives:
1. **PVC with `ReadOnlyMany`** (e.g., EFS/FSx on AWS, NFS, Azure Files) — model weights live on shared storage; any pod on any node mounts the same volume read-only. Slow first-load, simple ops.
2. **Init-container pulls from S3** (or HF hub) into an `emptyDir` at pod start — node-local for fast subsequent reads, but every cold start pays the download. Pair with image preloading or a sidecar puller cache.

KServe (Phase 1c) leans on the second pattern via its `storageUri` abstraction. Phase 2's GPU node group will probably want option 1 to avoid re-downloading 16 GB weights per pod start.

### Gotchas hit
- `hostPath` with `type: Directory` requires the dir to **already exist on the node**. `docker cp` to a non-existent dest path doesn't auto-create the parent — `docker exec helios-control-plane mkdir -p /models` first, then `docker cp`. Skipping the `mkdir` produces a confusing "Successfully copied / Could not find /models" pair.
- First `kubectl apply` will sit in `ImagePullBackOff` for a while as the kind node pulls `ghcr.io/ggml-org/llama.cpp:server`. Patience, not panic.

## Gotchas (from the PHASE_1.md checklist + what to watch for)
- **Port 8080 already in use** → map a different host port (`-p 8088:8080`).
- **HF download stalls** → `curl -C -` to resume.
- **Garbled output** → wrong chat template. Try `--chat-template qwen` or a different GGUF build.

## When I'd pick this vs alternatives
- **Pick llama.cpp server when**: laptop/CPU dev, tiny models, you just need an OpenAI-API-shaped endpoint to iterate against.
- **Don't pick when**: you have a real GPU and care about throughput — use vLLM (PagedAttention + continuous batching win here). At that point, swap the image; the API stays the same.

## Links
- llama.cpp server docs: https://github.com/ggml-org/llama.cpp/tree/master/examples/server
- Phase 1 walkthrough: [../PHASE_1.md](../PHASE_1.md)
- Related notes: [00-glossary.md](./00-glossary.md) — TTFT, TPOT, GGUF, KV cache, OpenAI-compatible API