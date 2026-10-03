# Demo shot list — 60–90 seconds, no voiceover (captions suffice)
# Record with any screen recorder (e.g. QuickTime). Keep the terminal large.

## Shot 1 — The parked state (0:00–0:10)
`kubectl get nodes` → only the e2-medium CPU node.
Caption: "Parked. The GPU costs nothing right now."

## Shot 2 — Scale up (0:10–0:25)
`kubectl scale deploy vllm --replicas=1`
`kubectl get pods -w` → Pending → ContainerCreating → Running, 0/1.
Caption: "One command. The autoscaler provisions an L4."

## Shot 3 — The GPU arrives (0:25–0:35)
`kubectl get nodes` → gpu-pool node appears, Ready.
Caption: "A $/hr GPU, billed per second, only while it works."

## Shot 4 — Probes doing their job (0:35–0:45)
Pod sits at 0/1 while the model loads (~3 min — timelapse this).
Then flips to 1/1.
Caption: "'Running' meant nothing until /health passed."

## Shot 5 — Inference (0:45–1:05)
`kubectl port-forward svc/vllm 8000:8000` (second terminal),
then curl the completions endpoint — tokens stream back.
Caption: "First inference on the L4."

## Shot 6 — The dashboard (1:05–1:20)
Screen-record the Cloud Monitoring dashboard: queue depth, running requests,
KV-cache %, throughput widgets with live data.
Caption: "Queue depth, throughput, KV-cache — via Managed Prometheus."

## Shot 7 — Park it again (1:20–1:30)
`kubectl scale deploy vllm --replicas=0` → pod terminates.
Caption: "Done. Back to $0 GPU spend."

## End card
Repo URL + "CPU reported 22% at full GPU load. Scale on queue depth."
