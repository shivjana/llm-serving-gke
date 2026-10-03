# LinkedIn launch post (copy-paste ready)
# Post as NATIVE LinkedIn document (upload write-up PDF) — link posts get throttled.
# Put the GitHub repo URL in the first comment. Tue–Thu, 8–10am PT.

Production-style LLM inference on GKE: vLLM on NVIDIA L4 GPUs, with real observability and honest autoscaling.

The setup: vLLM 0.30.0 serving TinyLlama 1.1B (2048 tokens of context) on Google Kubernetes Engine in us-central1, powered by an NVIDIA L4 GPU — the chip that does the actual AI math. TinyLlama is a small open model, good for learning the ropes. First successful inference: 60 tokens back from the GPU. Built over about 8 days, a few evenings a week.

Three things make this more than a toy:

Serve: the GPU hires itself only when needed. The GPU node pool (a g2-standard-4 machine with one L4) autoscales 0 to 1 — Kubernetes creates it when I deploy the model and deletes it when I scale to zero. No expensive hardware sitting idle overnight, and a $20/month budget alert keeps me honest. Getting here took two real fixes: Kubernetes silently injects service environment variables into every pod, which poisoned vLLM's port config (fixed with enableServiceLinks: false), and rolling updates deadlock on a single GPU because the new pod needs the GPU before the old one releases it (fixed with a Recreate strategy).

Harden: honest health checks. Loading the model into the GPU takes about 3 minutes, so I added three probes against vLLM's /health endpoint — a startup probe with a 5-minute budget, a readiness probe that gates traffic, and a gentle liveness probe that never fires during slow generations. I watched the new pod sit at 0/1 for the full 3-minute load, then flip to 1/1 the moment /health passed. Before this, it would have claimed "Running" from the first second.

Observe: a real dashboard. A PodMonitoring resource tells Google's Managed Prometheus to scrape the server's /metrics endpoint every 30 seconds, feeding a 4-widget Cloud Monitoring dashboard: queue depth, running requests, KV-cache pressure, token throughput. Two gotchas I earned the hard way: Google preserves the colon in metric names (vllm:num_requests_waiting, not vllm_num_requests_waiting — that wrong guess cost me 20 minutes), and pod restarts zero the token counter, which makes the throughput graph show a fake spike.

The finding worth your time: at full inference load — six parallel request loops driving pod CPU from 11 to 171 millicores — the CPU-based autoscaler reported 22% of its 70% target. The GPU was saturated. CPU is the wrong scaling signal for LLM serving because the heavy math happens on the GPU — queue depth (waiting requests) is the honest signal. I designed the custom-metrics pipeline for it (adapter → Kubernetes custom metrics API → HPA targeting 5 waiting requests per pod), then deliberately left it switched off: with a single-GPU quota, a second replica would sit Pending forever. Documented constraints beat theatrical demos.

Being honest about scope: one small model, one GPU, no public web address — you reach it through a secure tunnel (kubectl port-forward). This is a learning platform, not a production service. The repo has everything: the deployment files, the dashboard queries, and the autoscaling design clearly labeled for what it is.

Repo + full write-up in the comments. Feedback welcome — particularly on the autoscaling design and anything you'd do differently.

Question for the infra folks: what's a metric you trusted that turned out to be lying to you?

#Kubernetes #GCP #MLOps #LLMInference #AIPlatform
