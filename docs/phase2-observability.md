# LLM Serving on GKE — Phase 2: Production Hardening & Observability

**Project:** `oval-flow-350521` · GKE Standard (`llm-serving`, us-central1)
**Workload:** vLLM 0.30.0 serving TinyLlama/TinyLlama-1.1B-Chat-v1.0 on 1× NVIDIA L4
**Session date:** 2026-10-02
**Status labels used in this doc:** ✅ completed · 🟡 designed, not activated

Phase 1 (2026-09-25 → 09-29) built the serving stack, ran first inference on the L4,
and characterized CPU-based HPA behavior. This document covers Phase 2: health probes,
managed Prometheus scraping, the Cloud Monitoring dashboard, and the custom-metric
autoscaling design.

---

## 1. Health probes ✅

**Problem.** vLLM takes minutes to load the model into GPU memory after the container
starts. The deployment had no probes, so a pod reported `1/1 Running` from the first
second — Kubernetes could not distinguish "container started" from "model ready to
serve." A slow image pull or model load could also get the container killed as
unhealthy mid-startup.

**Change.** Strategic merge patch on the deployment adding three HTTP probes against
vLLM's `/health` endpoint (port 8000):

| Probe | Question it answers | Config |
|---|---|---|
| `startupProbe` | Is the model still loading? | `periodSeconds: 10`, `failureThreshold: 30` → 5-min budget; gates the other two probes |
| `readinessProbe` | Ready for traffic? | `periodSeconds: 10`, `failureThreshold: 3` |
| `livenessProbe` | Permanently wedged? | `periodSeconds: 30`, `failureThreshold: 3` → 90 s before restart; gentle enough that slow generations never trigger it |

**Verification.** The Recreate strategy rolled a new pod (`vllm-8469c44ddc-b72rt`).
Observed via `kubectl get pods -w`: the pod held at **0/1 for ~3 minutes** during
model load (startup probe doing its job), then flipped to **1/1** the moment
`/health` passed. Before this change it would have shown 1/1 throughout — the probes
turned "Running" into a meaningful "Ready."

---

## 2. Managed Service for Prometheus ✅

**Approach.** No self-managed Prometheus server. A `PodMonitoring` resource declares
the scrape target; Google's managed collector (DaemonSet in `gmp-system`, one per
node) does the scraping and writes into Cloud Monitoring.

```yaml
apiVersion: monitoring.googleapis.com/v1
kind: PodMonitoring
metadata:
  name: vllm
spec:
  selector:
    matchLabels:
      app: vllm
  endpoints:
  - port: 8000
    path: /metrics
    interval: 30s
```

**Verification (end-to-end).**
1. Source side: port-forward + `curl /metrics` confirmed vLLM exposes
   `vllm:num_requests_waiting`, `vllm:num_requests_running`, etc.
2. Pipeline side: `metricDescriptors` API confirmed the metrics landed in Cloud
   Monitoring (25+ `prometheus.googleapis.com/vllm:*` descriptors).

**Gotcha worth documenting.** Cloud Monitoring **preserves the colon** in the
metric type: `prometheus.googleapis.com/vllm:num_requests_waiting/gauge` — not
`vllm_num_requests_waiting` with an underscore. An incorrect underscore assumption
cost ~20 minutes of debugging (404s from the API while data was flowing fine the
whole time). Lesson: verify names via the descriptors API; never guess
normalization.

---

## 3. Cloud Monitoring dashboard ✅

Built declaratively via `gcloud monitoring dashboards create --config-from-file`
(four `xyChart` widgets, 2-column grid):

| Widget | Metric | Aggregation |
|---|---|---|
| Requests waiting (queue depth) | `vllm:num_requests_waiting/gauge` | 60 s mean |
| Requests running | `vllm:num_requests_running/gauge` | 60 s mean |
| KV cache usage % | `vllm:kv_cache_usage_perc/gauge` | 60 s mean |
| Generation throughput (tokens/sec) | `vllm:generation_tokens_total/counter` | `ALIGN_RATE` over 60 s |

**Verification.** All four widgets rendered with live data in the console.

**Observation.** A throughput spike appeared that was **not real traffic** — it was a
counter-reset artifact: the probe-rollout pod restart zeroed
`generation_tokens_total`, and the rate calculation across the reset renders as a
burst. Lesson for production dashboards: treat `rate()` skeptically around
restarts/deployments.

---

## 4. Custom-metric autoscaling design 🟡

**The signal.** `vllm:num_requests_waiting` (queue depth). Phase 1 proved CPU is the
wrong trigger for this workload: under six parallel inference loops, pod CPU rose
11m → 171m while the HPA reported only 22%/70%. Queue depth rises exactly when
requests pile up — it is the honest autoscaling signal for GPU inference.

**The architecture.**
1. Google's `custom-metrics-stackdriver-adapter` bridges Cloud Monitoring →
   Kubernetes' `custom.metrics.k8s.io` API (HPA cannot read Cloud Monitoring
   directly).
2. HPA with `type: Pods` (per-pod average, not utilization):

```yaml
metrics:
- type: Pods
  pods:
    metric:
      name: vllm_requests_waiting
    target:
      type: AverageValue
      averageValue: 5
```

**Why not activated.** The binding chain measured live on 2026-09-29 is unchanged:
regional L4 quota (1) < node-pool max size (1) < HPA max replicas (3). A
queue-driven HPA would correctly demand a second replica under load — and that pod
would sit `Pending` forever (`Insufficient nvidia.com/gpu`, autoscaler capped at
max node group size). Activating it would be theater. This item is honestly labeled
**designed, pending GPU quota headroom** — the design is complete and the
constraint is experimentally verified, which is the substantive engineering result.

---

## 5. Cost accounting (this session)

- L4 (`g2-standard-4`) active ~50 minutes for the session; deployment scaled back to
  0 and the GPU node left to drain afterward.
- Steady state: 1× `e2-medium` CPU node + negligible managed-metrics ingestion.
- No new standing spend introduced by Phase 2 (probes, PodMonitoring, dashboard
  are all control-plane/config; metrics ingestion at this volume is negligible).

---

## 6. Lessons

1. **Probes make "Ready" meaningful.** For model servers with minute-scale load
   times, `1/1 Running` without probes is a lie Kubernetes tells itself.
2. **Verify metric names via the API.** Normalization guesses (colon → underscore)
   cost real debugging time; the descriptors list is ground truth.
3. **Counters lie across restarts.** Rate graphs need reset-aware reading after
   every rollout.
4. **Label honest scope.** "Designed, not activated (binding constraint: GPU
   quota)" is a stronger engineering statement than a silently broken autoscaler.
5. **Debugging order for managed Prometheus:** source (`/metrics` via
   port-forward) → operator logs → collector config/discovery → Cloud Monitoring
   API. Check each link; don't assume.
