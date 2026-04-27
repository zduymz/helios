# 02 — KServe (`InferenceService`) on Knative

## What it is (1 sentence)
A Kubernetes operator that defines `InferenceService` — a CRD that wraps a model server with autoscaling, scale-to-zero, canary traffic split, and a standard predictor/transformer/explainer shape — leaning on Knative Serving + Istio under the hood (in Serverless mode).

## Why it exists (the problem it solves)
The 1b raw `Deployment` + `Service` works, but every team that wants to deploy a model has to re-invent: replicas, HPA tuning, ingress, canary, scale-to-zero, blue/green. `InferenceService` is the standardized "unit of a deployed model" — one CRD that's also the right thing to target from Backstage scaffolders later (Phase 8). KServe also separates **predictor** (the model server) from **transformer** (pre/post-processing) and **explainer** (interpretability), which keeps the model-serving boundary clean.

## Key concepts
- **`InferenceService` CRD** — top-level resource. Owns one or more of: `predictor`, `transformer`, `explainer`. Custom predictor = "run my container, KServe just gives me autoscaling + ingress + canary".
- **Deployment modes**:
  - **`Serverless`** (annotation `serving.kserve.io/deploymentMode: "Serverless"`) — Knative-backed. Get scale-to-zero, traffic split between revisions, request-buffering during cold start. What we picked.
  - **`RawDeployment`** — plain `Deployment` + HPA, no Knative. Use when scale-to-zero isn't worth the Knative complexity (always-on critical-path serving).
- **Knative Serving pieces under the hood**:
  - **`activator`** — buffers requests when target has 0 pods; spins up a pod, then forwards. Cold-start path.
  - **`autoscaler`** — collects concurrency/RPS metrics, decides desired replica count. Default scale-down delay = `stable-window` (60s) + `scale-to-zero-grace-period` (30s) ≈ 90s of idle.
  - **`queue-proxy`** — sidecar in every revision pod. Counts in-flight requests, exposes them to the autoscaler, and starts routing to the user container as soon as its TCP port opens (NOT when kubelet says `Ready`).
  - **`net-istio-controller`** — translates Knative `Ingress` → Istio `VirtualService` automatically. The `http://llama.helios.example.com` URL on `kubectl get isvc` is virtual; you reach it by hitting `istio-ingressgateway` with the right `Host:` header.
- **Concurrency metric** — Knative's default autoscaling signal. `scaleTarget: N` = "aim for N concurrent in-flight requests per pod". Alternative: `rps`.
- **Panic mode** — when actual concurrency exceeds 2× target during a burst, Knative jumps replicas aggressively (instead of waiting for the stable window). Why we saw 2 pods spawn for a single request at `scaleTarget: 1`.

## Commands I actually ran (Phase 1c)
```bash
# 1. Install: clone + run quick_install.sh (default: Serverless mode)
git clone --depth 1 --branch release-0.17 https://github.com/kserve/kserve.git /tmp/kserve-0.17
cd /tmp/kserve-0.17 && ./hack/quick_install.sh
# Installs: Istio 1.27, cert-manager 1.17, Knative 1.21 (via operator), KServe 0.17.

# 2. Allow hostPath volumes in Knative (operator-managed, so patch the CR not the configmap)
kubectl patch knativeserving knative-serving -n knative-serving --type merge \
  -p '{"spec":{"config":{"features":{"kubernetes.podspec-volumes-hostpath":"enabled"}}}}'

# 3. Apply the InferenceService
kubectl apply -f platform/serving/llama-cpp-kserve.yaml
kubectl get isvc llama -n helios -w   # wait for READY=True

# 4. Hit it through the Istio ingress gateway with the assigned Host header
kubectl -n istio-system port-forward svc/istio-ingressgateway 8090:80 &
curl -s -H "Host: llama.helios.example.com" \
  http://localhost:8090/v1/chat/completions \
  -H 'Content-Type: application/json' \
  -d '{"model":"qwen","messages":[{"role":"user","content":"hi"}]}' | jq

# 5. Observe scale-to-zero
kubectl get pods -n helios -w
# After ~90s idle: llama-predictor-* terminates. Next request → cold start.
```

## Configs / snippets I keep coming back to
The full manifest: [`platform/serving/llama-cpp-kserve.yaml`](../platform/serving/llama-cpp-kserve.yaml). Key fields:
```yaml
spec:
  predictor:
    minReplicas: 0          # scale-to-zero
    maxReplicas: 2
    scaleTarget: 1
    scaleMetric: concurrency
    containers:
      - name: kserve-container        # MUST be this name; KServe finds it by it
        ports:
          - name: http1               # Knative-required port name for HTTP/1.1 routing
            containerPort: 8080
        readinessProbe:               # if omitted, Knative uses TCP probe via queue-proxy
          httpGet: { path: /v1/models, port: 8080 }
```

## Numbers from my run (kind, single control-plane, model file cached on node)
- Install (KServe + deps via quick_install): **~3 min** wall-clock.
- `kubectl apply isvc` → `READY=True`: **~70 s** (Knative reconcile + first pod up).
- Idle → 0 pods: **~90 s** after last request (matches `stable-window + grace-period` = 60+30s).
- **Cold start (request blocked until first byte): ~2.4 s** for qwen-0.5B-Q4_K_M.

Compare to 1b's always-on rollout: ~12 s pod-Ready time. Cold start is *faster* than initial rollout because (a) the model file is in the OS page cache and (b) queue-proxy starts proxying on TCP-open, not kubelet `Ready` (which still waits the readiness probe's `initialDelaySeconds: 10`).

The honest version of "cold start" at production scale will be much longer — Llama-3.1-8B on a real GPU is 10–60s of weight load. Phase 1d will give us that number for real.

## Gotchas I hit
- **macOS bash 3.2 breaks `quick_install.sh`** → `readarray: command not found` at the very last step. Fix: install GNU bash via `brew install bash`, OR (what I did) install the missing `kserve-resources` chart manually:
  ```bash
  helm upgrade -i kserve-resources oci://ghcr.io/kserve/charts/kserve-resources \
    --version v0.17.0 --namespace kserve --wait --set kserve.version=v0.17.0
  ```
- **`quick_install.sh` is not idempotent on Istio** → re-running after a partial install hits `cannot re-use a name that is still in use` because the Istio install uses `helm install` (not `upgrade -i`). Either fully uninstall (`./hack/quick_install.sh -u`) and retry, or skip ahead to the failed chart.
- **`HostPath volume support is disabled`** webhook error from Knative → Knative locks down pod-spec by default; allowed fields are gated behind feature flags. Patch the **`KnativeServing`** CR (operator-managed install reverts plain ConfigMap patches), not `config-features` directly.
- **Pod shows `1/2 Ready` while curl returns 200** → queue-proxy is the second container; it routes traffic as soon as the user container's TCP port is up, decoupled from kubelet's readiness state. Useful: cold start in practice ≪ readiness probe `initialDelaySeconds`. Less useful: `kubectl get pods` Ready column underreports actual serving status.
- **Two pods spin up for one request at `scaleTarget: 1`** → panic mode. Burst-concurrency > 2× target during scale-from-zero triggers aggressive scale-up. Tune `scaleTarget: 5+` if pod-count surprises matter.
- **Status flapping during reconcile** → noisy `Operation cannot be fulfilled ... the object has been modified` events while KServe + Knative race to update the same object. Self-resolves; ignore.

## When I'd pick this vs alternatives
- **Pick `InferenceService` (Serverless) when**: cost-sensitive (GPUs idle 80% of the time), traffic is bursty, you want canary out of the box, the platform team wants one CRD to template against in Backstage.
- **Pick `InferenceService` (RawDeployment) when**: latency SLO can't tolerate cold start (e.g., 50ms p99), or Knative's pod-spec restrictions conflict with the workload (some GPU device plugins, certain volume types). You give up scale-to-zero and revision traffic split.
- **Pick raw `Deployment` (Phase 1b style) when**: prototyping, learning, or one-off internal services where the operational surface of Knative+Istio isn't justified.

## Links
- KServe docs: https://kserve.github.io/website/
- Knative autoscaling reference: https://knative.dev/docs/serving/autoscaling/
- `quick_install.sh` (release-0.17): https://github.com/kserve/kserve/blob/release-0.17/hack/quick_install.sh
- Manifest: [`../platform/serving/llama-cpp-kserve.yaml`](../platform/serving/llama-cpp-kserve.yaml)
- Decision: [`decisions/0001-kserve-over-raw-deployment.md`](./decisions/0001-kserve-over-raw-deployment.md)
- Glossary terms: [`00-glossary.md`](./00-glossary.md) — InferenceService, activator, queue-proxy, panic mode, concurrency metric.