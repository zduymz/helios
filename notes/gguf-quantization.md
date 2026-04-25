# GGUF & quantization

## What it is (1 sentence)
GGUF is a single-file container for quantized LLM weights + tokenizer + metadata, consumed by llama.cpp and its ecosystem; "quantized" means those weights are stored at fewer bits than their native FP16, trading a small accuracy loss for big memory and speed wins.

## Why it exists (the problem it solves)
An 8B-parameter model at native FP16 is ~16 GB — won't fit in laptop RAM, and matmuls over 16 GB of FP16 are too slow on CPU anyway. Quantization shrinks the weights to 4–8 bits each; GGUF bundles those quantized weights with everything a loader needs (arch, tokenizer, chat template) into one portable file. Together they're the reason "run a modern LLM on commodity hardware" is a thing at all.

## Key concepts
- **Weight** — a single learned parameter. An "8B model" has 8 billion of them.
- **Native precision** — how weights are held during training: FP16 or BF16 (2 bytes each). Baseline for memory math.
- **Quantization** — mapping FP16 weights to lower-bit representations (INT8, INT4, …). Slight accuracy cost, major size/speed gain.
- **Bits per weight (bpw)** — the main dial. ~8 bpw ≈ lossless, ~4 bpw ≈ best size/quality sweet spot, ~2 bpw ≈ visibly degraded.
- **K-quants (`_K` suffix)** — mixed-precision schemes: attention tensors get more bits, FFN gets fewer. Strictly better quality/bit than the legacy uniform quants (`Q4_0`, `Q8_0`, …).
- **S / M / L variants** — within a K-tier, Small / Medium / Large shift a few tensors up a bit. `_M` is the normal pick.
- **GGUF metadata** — architecture, context-length cap, RoPE params, chat template — all baked into the file so loaders can self-configure.
- **GGML (predecessor)** — older format from the same project, obsolete. `.bin` / `.ggml` files from before Aug 2023 won't load in current llama.cpp.

## The quant ladder (approximate for an 8B model)

FP16 baseline: ~16 GB (unquantized reference).

| Suffix | ~bpw | ~Size | Quality | Pick when |
|---|---|---|---|---|
| `Q8_0` | 8 | 8.5 GB | Near-lossless | RAM to spare |
| `Q6_K` | 6.5 | 6.6 GB | Excellent | Comfortable middle |
| `Q5_K_M` | 5.5 | 5.7 GB | Very good | Quality matters, have RAM |
| **`Q4_K_M`** | 4.8 | 4.9 GB | Good | **Community default** |
| `Q3_K_M` | 3.9 | 4.0 GB | Noticeable drop | Tight RAM |
| `Q2_K` | 3.0 | 3.2 GB | Rough | Last resort |

Rule of thumb: start at `Q4_K_M`. Step up to `Q5_K_M` if quality matters and RAM allows; step down only when forced.

## Decoding a filename

```
Qwen2.5-0.5B-Instruct-Q4_K_M.gguf
└──┬──┘ └┬─┘ └───┬───┘ └──┬──┘ └┬┘
   │     │       │        │    format
   │     │       │        └ 4-bit K-quant, Medium variant
   │     │       └ instruction-tuned (not base)
   │     └ 0.5 billion parameters
   └ model family + version
```

## Commands I actually ran
```bash
# Download (resumable; huggingface-cli works too)
curl -L -C - -o qwen2.5-0.5b-instruct-q4_k_m.gguf \
  https://huggingface.co/bartowski/Qwen2.5-0.5B-Instruct-GGUF/resolve/main/Qwen2.5-0.5B-Instruct-Q4_K_M.gguf

# Sanity-check: first 4 bytes should spell "GGUF"
head -c 4 qwen2.5-0.5b-instruct-q4_k_m.gguf; echo

# Compare sizes across quants of the same model
du -h *.gguf
```

## Gotchas to watch for
- **OOM even though the file fits** — you also have to budget RAM for the KV cache; it scales with `-c` (context length). File size is just the weights.
- **Garbled output** — chat template missing or wrong in the GGUF. Override with `--chat-template <name>`.
- **`.bin` / `.ggml` files** — pre-GGUF legacy, won't load in current llama.cpp. Find a re-quantized GGUF on HF.
- **`Q4_0` vs `Q4_K_M`** — `Q4_0` is legacy uniform 4-bit; `Q4_K_M` is the K-quant. Always prefer `_K_*` when both exist.
- **`IQ*` quants** — newer i-quants (e.g., `IQ4_XS`) squeeze more quality at 3–4 bpw using importance matrices. Slower per token on pure CPU; pick K-quants for CPU, i-quants when you want smaller files at similar quality on GPU.

## When I'd pick GGUF vs alternatives
- **Pick GGUF when**: CPU inference, laptop dev, quick local iteration, one-file convenience.
- **Don't pick when**: production GPU serving with high concurrency. Use raw FP16 / FP8, AWQ, or GPTQ on vLLM — PagedAttention + continuous batching assume contiguous tensor layouts that GGUF's packed quant blocks don't map onto as cleanly.

## Links
- GGUF spec: https://github.com/ggml-org/ggml/blob/master/docs/gguf.md
- llama.cpp quantize docs: https://github.com/ggml-org/llama.cpp/blob/master/examples/quantize/README.md
- Bartowski (well-maintained GGUF converts on HF): https://huggingface.co/bartowski
- Related notes: [00-glossary.md](./00-glossary.md), [01-inference-serving.md](./01-inference-serving.md)
