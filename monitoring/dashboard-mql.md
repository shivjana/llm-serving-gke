# Dashboard MQL — 4 widgets (2-column grid)
# Built via: gcloud monitoring dashboards create --config-from-file
# Paste each query into a Cloud Monitoring xyChart widget.

## 1. Requests waiting (queue depth) — gauge, 60s mean
fetch prometheus_target
| metric 'prometheus.googleapis.com/vllm:num_requests_waiting/gauge'
| group_by 1m, [value_num_requests_waiting_mean: mean(value.num_requests_waiting)]
| every 1m

## 2. Requests running — gauge, 60s mean
fetch prometheus_target
| metric 'prometheus.googleapis.com/vllm:num_requests_running/gauge'
| group_by 1m, [value_num_requests_running_mean: mean(value.num_requests_running)]
| every 1m

## 3. KV-cache usage % — gauge, 60s mean
fetch prometheus_target
| metric 'prometheus.googleapis.com/vllm:kv_cache_usage_perc/gauge'
| group_by 1m, [value_kv_cache_usage_perc_mean: mean(value.kv_cache_usage_perc)]
| every 1m

## 4. Generation throughput (tokens/sec) — counter, ALIGN_RATE over 60s
fetch prometheus_target
| metric 'prometheus.googleapis.com/vllm:generation_tokens_total/counter'
| align rate(1m)
| every 1m

# NOTE on widget 4: pod restarts zero the counter, and the rate across the reset
# renders as an artificial spike. Read rate() skeptically around rollouts.
