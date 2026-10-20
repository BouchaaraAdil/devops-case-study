# Runbooks

One section per alert. Each alert links here through its `runbook_url` annotation, and the Slack message shows the link. The kube-prometheus-stack default rules link to [runbooks.prometheus-operator.dev](https://runbooks.prometheus-operator.dev/) instead.

For any app alert, I start the same way:

1. Open Grafana, Service Overview (SLO), and pick the service. Is budget burning? Which golden signal moved? Is there a deploy marker nearby?
2. On the latency or error panel, click an exemplar dot and open the trace. It shows which span holds the time or the error: the app itself, a psycopg2 query, or a downstream call.
3. On that span, "Logs for this span" opens the request's log lines in Loki.
4. In Kubernetes, check `kubectl -n <ns> describe pod`, the events, restarts and the last termination reason.

Commands below assume `kubectl --context k3d-devops-cs`.

## Platform

### ServiceDown

**Meaning:** ml-api, backend-api or postgres has had 0 available pods for 1m. Users get errors or connection refused.

**Check:**
- `kubectl -n <ns> get pods` and `describe pod`. Look for CrashLoopBackOff, OOMKilled, an image pull error, or NotReady.
- NotReady with no restarts usually means `/ready` fails. backend-api's `/ready` needs a DB connection, so a full pool or a dead Postgres takes both pods out. See [BackendDbPoolExhausted](#backenddbpoolexhausted).
- The SLO burn alerts may stay quiet. With no ready pod, no request reaches the app, so its SLI sees no traffic.

**Fix:** fix whatever `describe` shows. A rollout restart buys time for a leak. If it started with a deploy, roll back.

### ServiceDegraded

**Meaning:** fewer pods are available than desired for 5m. The service still answers, with less capacity.

**Check:** `kubectl -n <ns> get pods,rs`. Look for a stuck rollout, a pod stuck in Pending (not enough resources), or one pod failing readiness.

**Fix:** depends on the pod state. If the new ReplicaSet is the one failing, `kubectl rollout undo`.

### ScrapeTargetDown

**Meaning:** the otel-agent hasn't been able to scrape a pod's `/metrics` for 2m. The pod is running but `/metrics` doesn't answer.

**Check:** `up{job=~"ml-api|backend-api"}` per pod, then `kubectl -n <ns> exec <pod> -- curl -s localhost:8000/metrics | head`. The agent logs help too: `kubectl -n observability logs ds/otel-agent-collector | rg scrape`.

**Fix:** usually the app is hung or overloaded, so restart the pod. If every target is down at once, look at the agent and the Target Allocator instead.

### AppMetricsMissing

**Meaning:** there's been no `up` series for ml-api or backend-api for 5m. In practice it fires about 10m after the cause, because Prometheus keeps the last sample for 5m. App metrics are pushed by the collectors, so when the collectors stop, the series disappear instead of going to 0.

**Check:**
- `kubectl -n observability get pods -l app.kubernetes.io/component=opentelemetry-collector`
- The agent and gateway logs. If [OtelExportFailing](#otelexportfailing) fires on the agent, the gateway is the problem.
- Traces and logs stop as well when the collectors are down.

**Fix:** restart or fix the collector. The agents keep a disk queue and replay it once the gateway is back, so part of the gap fills in.

### OtelExportFailing

**Meaning:** a collector exporter has failed to send for 5m. The `exporter` label tells you which backend: `otlp_grpc/gateway` (agent to gateway), `otlp_http/prometheus`, `otlp_http/loki` or `otlp_grpc/tempo`. The other signals keep flowing.

**Check:** that backend's pod (`kubectl -n observability get pods`) and the collector logs (`kubectl -n observability logs deploy/otel-gateway-collector | rg -i error`). Prometheus rejects out-of-order or too-old samples with HTTP 400. Loki rejects lines over its limits with 429.

**Fix:** bring the backend back, or fix the data it rejects. The export queue retries for a while. Data older than the retry window is lost.

### PodRestarting

**Meaning:** more than 2 restarts in 15m.

**Check:** `kubectl -n <ns> describe pod <pod>` and look at the `Last State` reason.
- `OOMKilled`: see [ContainerMemoryNearLimit](#containermemorynearlimit) and [MLApiMemoryGrowing](#mlapimemorygrowing).
- Liveness failures: in Loki, `{service_name="<svc>"} |= "/health" |= "503"`. Failing probe lines are kept, only the 200s are dropped.
- A crash: `kubectl logs <pod> --previous`.

**Fix:** depends on the reason. Note that a liveness restart loop can leave `/predict` working between restarts, so the error-rate alerts may stay quiet.

### ContainerMemoryNearLimit

**Meaning:** a container's working set has been above 85% of its memory limit for 5m. An OOMKill is likely next.

**Check:** the USE row on ML Services, memory vs limit. Is it a steady climb (a leak) or a step after a deploy (a new baseline)?

**Fix:** for a leak, see [MLApiMemoryGrowing](#mlapimemorygrowing). For a new baseline, raise the limit based on real usage.

## ml-api

### MLApiHighErrorRate

**Meaning:** more than 5% of `/predict` requests have returned 5xx for 5m.

**Check:** the errors panel on Service Overview, then an error trace and its logs. Tail sampling keeps every error trace. Did it start with a deploy marker?

**Fix:** roll back if it's a deploy. Otherwise follow the exception in the logs.

### MLApiHighLatency

**Meaning:** `/predict` p95 has been above 1s for 5m. Normal is about 400ms.

**Check:** from the latency panel, open an exemplar's trace. If the time sits in the ml-api server span with no child span, it's the app's own code, for example `RESPONSE_OVERHEAD_MS`. A long child span points at a dependency. CPU throttling on the USE row points at the CPU limit.

**Fix:** roll back or remove the cause. If it's load, scale out.

### MLApiNoPredictions

**Meaning:** `ml_api_predictions_total` hasn't moved for 10m. The pods may be up and passing probes while doing no real work.

**Check:** is traffic arriving (`ml_api_requests_total{endpoint="/predict"}`)? Is the load-generator running? Its logs show connection errors.

**Fix:** if the callers stopped, the problem is upstream. If requests arrive but no predictions are counted, look at the ml-api logs.

### MLApiMemoryGrowing

**Meaning:** `ml_api_memory_bytes` has grown steadily for 10m and is past 50MB. That looks like a leak. It usually goes on to ContainerMemoryNearLimit, then an OOMKill, then PodRestarting.

**Check:** on ML Services, `ml_api_memory_bytes` shows a sawtooth: it climbs, then drops at each restart. `kubectl -n ml-api describe pod` shows `Last State: OOMKilled`. Check the env for `MEM_ALLOC_MB`.

**Fix:** roll back or remove the cause. A restart only resets the clock. Longer term, profile the memory.

### MLApiAvailabilitySLOBurn

**Meaning:** the `/predict` availability SLO (99.5% over 30d) is burning budget too fast. `critical` is a fast burn that would use up the budget in about 2 days, so it pages. `warning` is a slow burn, worth a ticket.

**Check:** the SLO row on Service Overview, for budget left and the 1h/6h burn rate. Then the same steps as [MLApiHighErrorRate](#mlapihigherrorrate).

**Fix:** stop the errors first. Then look at the budget left and decide whether to freeze deploys.

### MLApiLatencySLOBurn

**Meaning:** too many `/predict` requests take over 500ms (SLO 99% over 30d). Same `critical` and `warning` split as above.

**Check:** same as [MLApiHighLatency](#mlapihighlatency). This one can fire before the p95 alert, because it counts every slow request.

**Fix:** same as MLApiHighLatency.

## backend-api

### BackendApiHighErrorRate

**Meaning:** more than 5% of `/process` requests have returned 5xx for 5m.

**Check:** open an error trace and see which span failed. If it's the psycopg2 span, see [BackendDbErrors](#backenddberrors). In Loki, `{service_name="backend-api"} |= "ERROR"`. If you see `relation "documents" does not exist` after a Postgres restart, the table is gone: Postgres runs on an `emptyDir`, and the backend only creates the table at startup.

**Fix:** for the missing table, restart backend-api with `kubectl -n backend-api rollout restart deploy/backend-api`. Otherwise roll back or follow the exception.

### BackendApiHighLatency

**Meaning:** `/process` p95 has been above 500ms for 5m. Normal p99 is about 10ms (one insert).

**Check:** open the `POST /process` trace from an exemplar. A long psycopg2 or SELECT span means the time goes to the database, for example `QUERY_OVERHEAD_MS`. Pool usage often rises at the same time, because slow queries hold connections longer.

**Fix:** fix the query or the Postgres side. Keep an eye on [BackendDbConnectionsHigh](#backenddbconnectionshigh).

### BackendDbPoolExhausted

**Meaning:** queries fail with `pool_exhausted`, or all 10 pool connections are in use. `/process` returns 503. Soon after, `/ready` fails too because it needs the pool, both pods go NotReady and the Service has no endpoints left. Liveness still passes, so nothing restarts on its own.

**Check:**
- Connections flat at 10 while traffic drops to 0 is a leak, not load. Under load, connections would follow traffic.
- `kubectl -n backend-api describe pod` shows `Readiness probe failed ... 503`.
- The load-generator logs show `Connection refused`, and its client spans have `STATUS_CODE_ERROR`.
- Check the env for `CONN_RETURN_MODE=hold`.
- This alert keeps firing while the pool is full. BackendApiHighErrorRate and the SLO burn alerts don't fire, because no request reaches the app.

**Fix:** `kubectl -n backend-api rollout restart deploy/backend-api` buys time. The real fix is returning connections to the pool, with a context manager or a `finally`.

### BackendDbConnectionsHigh

**Meaning:** a pod has held 8 or more of its 10 DB connections for 5m, so the pool is leaking or saturated. A fast leak reaches [BackendDbPoolExhausted](#backenddbpoolexhausted) first. This alert catches the slow ones.

**Check:** the DB pool panel on ML Services. Do connections follow traffic (saturation) or climb without it (a leak)? Check query latency too, since slow queries hold connections longer.

**Fix:** for a leak, restart and fix the code. For saturation, make the queries faster, add replicas, or grow the pool if Postgres `max_connections` allows it.

### BackendDbErrors

**Meaning:** more than 5% of DB queries have failed for 5m.

**Check:** is Postgres up (`kubectl -n postgres get pods`, see [ServiceDown](#servicedown))? Read the Postgres logs in Loki with `{service_name="postgres"}`, and the backend-api logs for the SQL error.

**Fix:** bring Postgres back. If it restarted, the `documents` table is gone (it's an `emptyDir`), so restart backend-api to recreate it.

### BackendApiAvailabilitySLOBurn

**Meaning:** the `/process` availability SLO (99.5% over 30d) is burning budget too fast. `critical` is a fast burn and pages, `warning` is a slow burn for a ticket.

**Check:** the SLO row on Service Overview, then the same steps as [BackendApiHighErrorRate](#backendapihigherrorrate).

**Fix:** stop the errors, then look at the budget left and decide whether to freeze deploys.

### BackendApiLatencySLOBurn

**Meaning:** too many `/process` requests take over 100ms (SLO 99% over 30d). It usually fires before [BackendApiHighLatency](#backendapihighlatency).

**Check:** same as [BackendApiHighLatency](#backendapihighlatency).

**Fix:** same as BackendApiHighLatency.
