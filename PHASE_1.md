# Phase 1 — Inference serving (vLLM + KServe)

**Skill**: deploy an LLM as a production-grade service on Kubernetes.

**Calendar**: ~1 week of evenings.

**Sub-phases**:
- **1a** (½ day) — LLM in Docker locally, understand the OpenAI-compatible API
- **1b** (1 day) — Same LLM in kind, as a Deployment + Service
- **1c** (1–2 days) — Wrap in KServe `InferenceService`, scale-to-zero
- **1d** (1 day, cost-gated) — Cloud GPU + vLLM + Llama-3.1-8B, benchmark, tear down

> **Pedagogy**: 1a–1c use **llama.cpp server** (runs on CPU) so you can iterate fast without burning cloud money. 1d swaps in **vLLM on a real GPU** so you feel the performance gap and understand *why* vLLM exists. This also teaches: the OpenAI API is the interface; implementations are swappable — the exact lesson LiteLLM monetizes in Phase 3.

---

## Before you start

- [ ] Docker Desktop (or colima) running
- [ ] `kind` installed (`brew install kind`)
- [ ] `kubectl` installed
- [ ] `helm` installed
- [ ] ~5 GB free disk for model weights
- [ ] A Hugging Face account + token (some models gate-access)

If you don't have a kind cluster yet, do **Phase 0** first — see [LEARNING_PATH.md](./LEARNING_PATH.md).

---

## Sub-phase 1a — LLM in Docker (½ day)

### Goal

A `curl` to `localhost:8080/v1/chat/completions` returns a real LLM response. You understand the OpenAI-compatible API surface end-to-end.

### Why llama.cpp server (not vLLM) for local

vLLM's CPU backend requires building from source with specific flags and is slow. `llama.cpp` runs GGUF-quantized models on CPU well, and exposes the **same OpenAI-compatible `/v1/chat/completions` API**. Phase 1d swaps it for vLLM on a GPU — same API, different implementation.

### Steps

**1. Pick a tiny model.** `Qwen2.5-0.5B-Instruct` in Q4 quantization is ~400 MB and runs fine on a laptop CPU.

```bash
mkdir -p ~/helios-models
cd ~/helios-models
curl -L -o qwen2.5-0.5b-instruct-q4_k_m.gguf \
  https://huggingface.co/bartowski/Qwen2.5-0.5B-Instruct-GGUF/resolve/main/Qwen2.5-0.5B-Instruct-Q4_K_M.gguf
```

**2. Run llama.cpp server in Docker.**

```bash
docker run -d --name helios-llm \
  -p 8080:8080 \
  -v ~/helios-models:/models \
  ghcr.io/ggml-org/llama.cpp:server \
  -m /models/qwen2.5-0.5b-instruct-q4_k_m.gguf \
  --host 0.0.0.0 --port 8080 \
  -c 4096
```

Flags: `-c 4096` is context length (KV cache size). Crank up if you can afford memory.

**3. Sanity-check the OpenAI API.**

```bash
# Models endpoint
curl -s localhost:8080/v1/models | jq

# Chat completion (note this is the standard OpenAI request shape)
curl -s localhost:8080/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "qwen2.5-0.5b",
    "messages": [{"role": "user", "content": "In one sentence, what is KV cache?"}]
  }' | jq '.choices[0].message.content'
```

**4. Measure TTFT (time-to-first-token) with streaming.**

```bash
time curl -sN localhost:8080/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{
    "model": "qwen2.5-0.5b",
    "stream": true,
    "messages": [{"role":"user","content":"Count from 1 to 20."}]
  }'
```

Note the time from request → first `data:` line in the SSE stream. That's TTFT.

### Verification

- [ ] `/v1/models` returns at least one model.
- [ ] A chat completion returns coherent English.
- [ ] You measured TTFT with streaming.

### Notes to capture (in this order)

Scaffold two files:

1. `notes/01-inference-serving.md` (will become `01-vllm.md` later when you swap to vLLM) — capture the concept of **OpenAI-compatible API as lingua franca**.
2. `notes/00-glossary.md` — add entries: **TTFT**, **TPOT**, **context length**, **GGUF**, **quantization** (Q4_K_M), **KV cache**.

Also: `notes/log/YYYY-MM-DD.md` — three bullets: did / broke / learned.

### Gotchas you might hit

- **Port clash**: 8080 is busy. Pick 8088 or similar.
- **Model download hangs**: the HF mirror sometimes rate-limits; use `curl -C -` to resume.
- **Response is nonsense**: wrong chat template for the model. Add `--chat-template qwen` or pick a different model build.

---

## Sub-phase 1b — LLM in kind as a Deployment (1 day)

### Goal

The same llama.cpp server, but now running in Kubernetes as a plain `Deployment` + `Service`. You can `kubectl port-forward` and hit it the same way.

### Steps

**1. Create a kind cluster (if you haven't).**

```bash
cat > infra/local/kind-config.yaml <<'EOF'
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 30080
        hostPort: 30080
        protocol: TCP
EOF

kind create cluster --name helios --config infra/local/kind-config.yaml
kubectl config use-context kind-helios
```

**2. Load the model into the cluster.** Two options:
- **Simple (dev only)**: bake the model into a container image. Fast local, but bloated image.
- **Better**: mount via a `hostPath` volume from the kind node.

Go with the hostPath approach (closer to how you'd do it in prod with a PVC):

```bash
# Copy the model into the kind node
docker cp ~/helios-models/qwen2.5-0.5b-instruct-q4_k_m.gguf \
  helios-control-plane:/models/qwen2.5-0.5b-instruct-q4_k_m.gguf
```

**3. Write the Deployment.**

```yaml
# platform/serving/llama-cpp-deployment.yaml
apiVersion: v1
kind: Namespace
metadata: { name: helios }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: llama-cpp
  namespace: helios
spec:
  replicas: 1
  selector: { matchLabels: { app: llama-cpp } }
  template:
    metadata: { labels: { app: llama-cpp } }
    spec:
      containers:
        - name: server
          image: ghcr.io/ggml-org/llama.cpp:server
          args:
            - "-m"
            - "/models/qwen2.5-0.5b-instruct-q4_k_m.gguf"
            - "--host"
            - "0.0.0.0"
            - "--port"
            - "8080"
            - "-c"
            - "4096"
          ports:
            - containerPort: 8080
          resources:
            requests: { cpu: "500m", memory: "1Gi" }
            limits:   { cpu: "2",    memory: "2Gi" }
          volumeMounts:
            - { name: models, mountPath: /models }
          readinessProbe:
            httpGet: { path: /v1/models, port: 8080 }
            initialDelaySeconds: 10
            periodSeconds: 5
      volumes:
        - name: models
          hostPath: { path: /models, type: Directory }
---
apiVersion: v1
kind: Service
metadata:
  name: llama-cpp
  namespace: helios
spec:
  type: NodePort
  selector: { app: llama-cpp }
  ports:
    - port: 8080
      targetPort: 8080
      nodePort: 30080
```

```bash
kubectl apply -f platform/serving/llama-cpp-deployment.yaml
kubectl -n helios rollout status deploy/llama-cpp
```

**4. Verify.**

```bash
# Via NodePort (kind forwards 30080 to host)
curl -s localhost:30080/v1/models | jq

# Via port-forward
kubectl -n helios port-forward svc/llama-cpp 8088:8080 &
curl -s localhost:8088/v1/models | jq
```

### Verification

- [ ] `kubectl get pods -n helios` shows `llama-cpp-xxxx` Running + Ready.
- [ ] Model endpoint works via NodePort AND port-forward.
- [ ] You can kill the pod and it restarts (`kubectl delete pod -l app=llama-cpp -n helios`).

### Notes to capture

- `notes/01-inference-serving.md`: why hostPath is dev-only (in prod → PVC + `ReadOnlyMany`, or an init-container that pulls from S3).
- `notes/log/...`: include "how long until the pod was Ready" — that's cold start without KServe. Compare against 1c.

### Gotchas

- **`ImagePullBackOff`**: kind needs to pull the image; first time is slow.
- **`CrashLoopBackOff` with "cannot open model"**: hostPath model didn't make it into the kind node. Re-run `docker cp`.
- **Readiness probe flapping**: bump `initialDelaySeconds`; llama.cpp takes 10–30s to load.

---

## Sub-phase 1c — KServe InferenceService (1–2 days)

### Goal

The same workload, but managed by KServe. You get: autoscaling, scale-to-zero, canary rollouts, and the `InferenceService` CRD as the standard unit of "a deployed model".

### Why KServe (not raw Deployment)

This becomes **ADR 0002**. Key reasons:
- **Scale-to-zero** (via Knative underneath) — critical when GPUs cost $1/hr.
- **Canary rollouts** built-in (`traffic` split between revisions).
- Standard CRD = golden-path friendly (Phase 8 Backstage templates target `InferenceService`).
- Clean separation of **predictor** (the model) vs **transformer** (pre/post-processing).

### Steps

**1. Install KServe.** Use the quick-install script for the serverless install (includes Knative + Istio + cert-manager):

```bash
curl -sL "https://raw.githubusercontent.com/kserve/kserve/release-0.13/hack/quick_install.sh" | bash
```

Wait for `kubectl get pods -A` to show everything Running.

**2. Define the InferenceService.** KServe's "standard" runtimes (sklearn, torchserve, etc.) don't natively serve a generic OpenAI API. You have two options:

- **Custom predictor** — tell KServe to run your llama.cpp container directly. Simplest.
- **Huggingface runtime** — KServe has a built-in HuggingFace server. Nice, but constrains model format.

Go with the custom predictor for learning; you'll see exactly what KServe does on top.

```yaml
# platform/serving/llama-cpp-kserve.yaml
apiVersion: serving.kserve.io/v1beta1
kind: InferenceService
metadata:
  name: llama
  namespace: helios
  annotations:
    serving.kserve.io/deploymentMode: "Serverless"   # Knative; enables scale-to-zero
spec:
  predictor:
    minReplicas: 0
    maxReplicas: 2
    scaleTarget: 1
    scaleMetric: concurrency
    containers:
      - name: kserve-container
        image: ghcr.io/ggml-org/llama.cpp:server
        args: ["-m","/models/qwen2.5-0.5b-instruct-q4_k_m.gguf","--host","0.0.0.0","--port","8080","-c","4096"]
        ports:
          - containerPort: 8080
            protocol: TCP
        resources:
          requests: { cpu: "500m", memory: "1Gi" }
          limits:   { cpu: "2",    memory: "2Gi" }
        volumeMounts:
          - { name: models, mountPath: /models }
    volumes:
      - name: models
        hostPath: { path: /models, type: Directory }
```

```bash
kubectl apply -f platform/serving/llama-cpp-kserve.yaml
kubectl -n helios get inferenceservice llama -w
```

**3. Hit it through Istio ingress.** KServe fronts predictors through Istio. Get the URL:

```bash
kubectl get isvc llama -n helios -o jsonpath='{.status.url}'
# → e.g., http://llama.helios.example.com
```

In kind, you need to forward the Istio gateway:

```bash
kubectl -n istio-system port-forward svc/istio-ingressgateway 8090:80 &

# Now hit it, providing the Host header
curl -s -H "Host: llama.helios.example.com" \
  http://localhost:8090/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"qwen","messages":[{"role":"user","content":"Hi"}]}' | jq
```

**4. Observe scale-to-zero.**

```bash
# Watch pods
kubectl -n helios get pods -w
```

Don't send traffic for ~90s. The pod will terminate. Then hit it again — see cold-start, pod comes back. **Measure the cold-start time.** Capture in notes.

### Verification

- [ ] `kubectl get isvc llama -n helios` shows `READY=True`.
- [ ] Requests route through Istio gateway.
- [ ] Zero pods after idle period; pod respawns on next request.
- [ ] You captured cold-start time.

### Notes to capture

- `notes/02-kserve.md` (the template it replaces is the `notes/01-inference-serving.md` scaffold): InferenceService CRD anatomy, predictor vs transformer, Serverless vs RawDeployment modes, Knative underneath, Istio VirtualService auto-gen.
- `notes/decisions/0002-kserve-over-raw-deployment.md`: the reasons above.
- `notes/00-glossary.md`: add **scale-to-zero**, **cold start**, **concurrency metric** (Knative's default autoscaling signal).

### Gotchas

- **`InferenceService` stuck in `Ready=False`**: check `kubectl describe isvc llama`. Common: Istio not ready, Knative serving CR missing, port name must be `http1` for some versions.
- **Istio Host header mismatch**: the URL KServe assigns is DNS-style; you *must* send the `Host:` header or configure `/etc/hosts`.
- **Scale-to-zero not triggering**: default scale-down delay is ~60s; be patient.

---

## Sub-phase 1d — Cloud GPU + vLLM + Llama-3.1-8B (1 day, cost-gated)

> **Cost warning**: an AWS `g5.xlarge` is ~$1/hr. Set a budget alarm at $50/mo **before starting**. Shut the instance down within the same session. Don't leave it running overnight.

### Goal

Feel the performance gap. Understand why vLLM matters at real model sizes.

### Options (cheapest first)

| Option | Cost | Notes |
|---|---|---|
| RunPod spot A10/L4 | ~$0.30/hr | Simplest; not integrated with your kind |
| AWS `g5.xlarge` (1× A10G, 24GB) | ~$1/hr | Closest to what AcmeCorp would run |
| GCP `g2-standard-8` (1× L4) | ~$0.70/hr | Good alt |

For real Phase 8 IDP integration you'll want EKS + GPU node group eventually. For Phase 1d, RunPod is fine — the point is to *run vLLM* and benchmark, not yet to wire cloud K8s.

### Steps (RunPod path, simplest)

1. Launch a RunPod container with image `vllm/vllm-openai:latest`, 1× A10 or L4, 24 GB.
2. Command: `--model meta-llama/Llama-3.1-8B-Instruct --max-model-len 8192 --gpu-memory-utilization 0.9`.
3. HTTP-expose port 8000.
4. Send the same OpenAI chat completion request. Note the response time.
5. Benchmark with the built-in vLLM benchmark script OR a simple loop:

```bash
# 20 concurrent requests, 50 total
for i in $(seq 1 50); do
  curl -s $VLLM_URL/v1/chat/completions \
    -H 'Content-Type: application/json' \
    -d '{"model":"meta-llama/Llama-3.1-8B-Instruct",
         "messages":[{"role":"user","content":"Summarize distributed consensus in 2 sentences."}]}' \
    -o /dev/null -w "%{time_total}\n" &
  [ $((i % 20)) -eq 0 ] && wait
done
```

6. Compare to llama.cpp on tiny model. Record:
   - TTFT at concurrency=1 vs concurrency=20
   - Total throughput (tokens/sec aggregated)
   - Peak GPU memory used

7. **Shut it down.** `terminate` on RunPod. Confirm.

### Verification

- [ ] Real 8B model answered a real prompt.
- [ ] Benchmarked at multiple concurrency levels.
- [ ] Instance terminated; confirmed no running pods in RunPod console.

### Notes to capture

- `notes/01-vllm.md` (promote from the earlier `01-inference-serving.md` scaffold): PagedAttention, continuous batching, `--max-model-len` vs `--gpu-memory-utilization` trade-off, quantization options (AWQ, GPTQ, FP8), speculative decoding.
- `notes/decisions/0002-*`: add an addendum — vLLM vs llama.cpp vs TGI — picked vLLM because of continuous batching + PagedAttention + the OSS ecosystem momentum.
- `notes/00-glossary.md`: add **PagedAttention**, **continuous batching**, **speculative decoding**, **AWQ**, **GPTQ**.
- `notes/incidents/YYYY-MM-DD-first-cloud-gpu-bill.md`: how much it cost you, whether you shut it down on time. Lessons.

### Gotchas

- **OOM on model load**: lower `--max-model-len`, then lower `--gpu-memory-utilization`. The KV cache eats most memory at long context lengths.
- **Tokenizer mismatch between your eval and the model**: always pin exact HF revision.
- **Leaving the instance running**: set a calendar reminder for 2 hours after launch.

---

## Phase 1 exit criteria

All of these true before moving on to Phase 2:

- [ ] llama.cpp server runs in kind via `InferenceService` with scale-to-zero.
- [ ] You've hit a real Llama-3.1-8B on a real GPU once, and shut it down.
- [ ] `notes/01-vllm.md`, `notes/02-kserve.md`, `notes/00-glossary.md` all populated.
- [ ] `notes/decisions/0002-kserve-over-raw-deployment.md` written.
- [ ] 3+ entries in `notes/log/`.
- [ ] Commit and push. Root `README.md` "Status" table updated: Phase 1 ☑.

Then: **[LEARNING_PATH.md](./LEARNING_PATH.md) → Phase 2** (GPU scheduling).
