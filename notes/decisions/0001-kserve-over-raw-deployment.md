# 0001 — KServe `InferenceService` (Serverless mode) over raw `Deployment` for model serving

Date: 2026-04-25
Status: Accepted

## Context
Phase 1b proved a llama.cpp server in kind as a plain `Deployment` + NodePort `Service` works. But the platform's job (per the project brief) is to give product teams a paved road for "deploy a model" — not to make every team re-implement autoscaling, ingress, canary, and scale-to-zero. We also need a CRD that's stable enough to be the template target in the Phase 8 Backstage IDP. Cost matters (cloud GPUs at $1/hr+); idle replicas are a real bill.

## Options considered
1. **Raw `Deployment` + HPA + Service/Gateway** — what 1b does. Familiar; no extra control plane.
2. **KServe `InferenceService` in Serverless mode** — Knative under the hood: scale-to-zero, revision-based traffic split, queue-proxy.
3. **KServe `InferenceService` in RawDeployment mode** — KServe CRD shape, but plain `Deployment` + HPA underneath; no Knative.
4. **Knative Service directly** (no KServe) — get scale-to-zero, but lose KServe's predictor/transformer/explainer shape and the model-serving CRD that Backstage will scaffold.

## Decision
Option 2: KServe `InferenceService` in **Serverless mode** as the golden-path "deployed model" abstraction.

## Why (the reasoning future-you needs)
- **Scale-to-zero is non-negotiable for GPUs.** A single idle 8B-model pod on an A10 burns ~$24/day. Knative gets us to zero pods after ~90s idle, with cold start observed at 2.4s for a tiny CPU model (will be 10–60s on GPU; still cheaper than always-on).
- **Canary out of the box.** Knative revisions + traffic-percent split are wired into `InferenceService` via `traffic:`. Phase 8 templates can expose "promote 10% to v2" as a one-line PR diff.
- **One CRD to scaffold.** Backstage golden paths are easier to template against `InferenceService` than against the (Deployment, Service, HPA, Gateway) tuple.
- **Predictor/transformer separation.** Pre/post-processing belongs in a separate container, not bolted into the model server. KServe's spec already has the seam; we'll use it in Phase 4+ for token-counting, prompt-injection scrubbing, etc.
- **OpenAI API survives unchanged.** Custom-predictor mode just runs our llama.cpp container. The interface from Phases 1a/1b is the same.

## Trade-offs accepted
- **Operational surface explodes.** Knative + Istio + cert-manager + KServe is ~4 control planes. We just paid this tax: Helm install dance, hostPath feature-flag patch, mac bash bug. In return we don't write autoscaling logic.
- **Knative pod-spec restrictions.** hostPath, hostNetwork, init containers, etc. are gated behind feature flags. Some GPU device-plugin patterns may need RawDeployment.
- **Panic-mode pod overshoot.** At low `scaleTarget` (1), a single request can briefly spawn 2 pods. Acceptable; tune `scaleTarget` as workload size grows.
- **Cold-start tail latency.** A scale-from-zero request blocks at the activator until the new pod's TCP port is up. Bad for sub-second SLOs. Mitigation: keep `minReplicas: 1` for latency-critical paths; accept zero for batch/long-running.
- **Knative learning curve.** Activator vs autoscaler vs queue-proxy is non-obvious; debugging stuck revisions is annoying. Worth it.

## Revisit when
- Cold-start P99 exceeds the product SLO for an interactive use case (e.g., chat). Then move that specific service to RawDeployment with `minReplicas: 1`, or switch to KEDA-driven scaling on a plain Deployment.
- Knative panic-mode pod overshoot causes GPU OOM (multiple 8B-model pods on one node). Then enforce `containerConcurrency` + tune `panic-window-percentage` / `panic-threshold-percentage`.
- KServe drops Serverless mode or moves to a v2 CRD shape with breaking changes — re-evaluate the entire serving stack.
- Backstage / IDP team picks a different scaffolding target — we'd want the golden path to align with whatever the IDP can template cleanly.

## Related
- Topic note: [`../02-kserve.md`](../02-kserve.md)
- Manifest: [`../../platform/serving/llama-cpp-kserve.yaml`](../../platform/serving/llama-cpp-kserve.yaml)
- 1b raw-Deployment baseline: [`../../platform/serving/llama-cpp-deployment.yaml`](../../platform/serving/llama-cpp-deployment.yaml)
