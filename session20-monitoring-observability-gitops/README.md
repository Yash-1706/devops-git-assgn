# Session 20: Monitoring, Observability and GitOps

**Name:** Yash Gaikwad  |  **Roll No:** 24BCS10100

| Task | Topic | Where |
|---|---|---|
| 1 | Monitoring: metrics, logs, alerts, CPU utilisation, memory utilisation, application health (demo) | [Task 1](#task-1-monitoring) |
| 2 | Observability: the three pillars, why it is needed, tools, Kubernetes observability | [Task 2](#task-2-observability) and [observability/README.md](observability/README.md) |
| 3 | GitOps: concepts and a hands-on Argo CD demo (also the session's mini project) | [Task 3](#task-3-gitops) and [gitops/README.md](gitops/README.md) |

**Deliverables:** monitoring demo ([`monitoring-demo/`](monitoring-demo)), observability documentation, GitOps demo ([`gitops/`](gitops)), screenshots, this README.

**Environment:** Windows 11 with Docker Desktop (Task 1 runs in plain Docker), Minikube (Docker driver, Kubernetes v1.37.0) with Argo CD for Task 3. All output and screenshots are real; the screenshots come from a headless Microsoft Edge pointed at the live services.

---

## Task 1: Monitoring

**Monitoring** means collecting numbers about a running system, showing them, and getting alerted when they look wrong. This demo covers the six things the task lists: **metrics, logs, alerts, CPU utilisation, memory utilisation and application health**.

### The setup

```
 curl / load ---> [ demo app :8000 ]  --/metrics--+
                  (Flask, JSON logs,              |
                   /healthz, /metrics)            v
 [ node-exporter ]  --/metrics----------->  [ Prometheus :9090 ] --alerts--> [ Alertmanager :9093 ]
 (host CPU, memory)                                |
                                                   v
                                            [ Grafana :3000 ]  (dashboard)
```

| Component | Role |
|---|---|
| **demo app** | my small Flask application: counts requests (`app_http_requests_total`), measures latency, exposes `app_healthy`, `/healthz` and `/metrics`, and writes **JSON logs** to stdout |
| **node-exporter** | CPU and memory of the host the containers run on |
| **Prometheus** | scrapes the metrics every 5 s, stores them, evaluates the **alert rules** |
| **Alertmanager** | receives firing alerts, groups them and routes them (placeholder receiver here) |
| **Grafana** | dashboard built from PromQL queries (provisioned from files) |

All files are in [`monitoring-demo/`](monitoring-demo/). Everything below was run for real with Docker Desktop.

_The alert rules: AppDown, AppUnhealthy, HighErrorRate (application) and HighCPUUsage, HighMemoryUsage (host). `for:` means the condition must hold that long before the alert fires._

```console
$ cat monitoring-demo/prometheus/alerts.yml
groups:
  - name: application
    rules:
      - alert: AppDown
        expr: up{job="demo-app"} == 0
        for: 15s
        labels:
          severity: critical
        annotations:
          summary: "demo-app is down (Prometheus cannot scrape it)"

      - alert: AppUnhealthy
        expr: app_healthy == 0
        for: 10s
        labels:
          severity: critical
        annotations:
          summary: "demo-app reports itself unhealthy (/healthz returns 503)"

      - alert: HighErrorRate
        expr: |
          sum(rate(app_http_requests_total{status=~"5.."}[30s]))
            / sum(rate(app_http_requests_total[30s])) > 0.2
        for: 10s
        labels:
          severity: warning
        annotations:
          summary: "More than 20% of the requests fail with a 5xx error"

  - name: host
    rules:
      - alert: HighCPUUsage
        expr: 100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[30s])) * 100) > 40
        for: 10s
        labels:
          severity: warning
        annotations:
          summary: "CPU utilisation is above 40%"

      - alert: HighMemoryUsage
        expr: (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100 > 85
        for: 30s
        labels:
          severity: warning
        annotations:
          summary: "Memory utilisation is above 85%"
```

_Start the whole stack (build my app image, start Prometheus, Alertmanager, node-exporter and Grafana)._

```console
$ cd monitoring-demo && docker compose up -d --build 2>&1 | tail -n 8
 Container s20mon-app Starting 
 Container s20mon-app Started 
 Container s20mon-node-exporter Started 
 Container s20mon-alertmanager Started 
 Container s20mon-prometheus Starting 
 Container s20mon-prometheus Started 
 Container s20mon-grafana Starting 
 Container s20mon-grafana Started 
```

_Five containers running._

```console
$ cd monitoring-demo && docker compose ps --format 'table {{.Service}}\t{{.Image}}\t{{.Status}}\t{{.Ports}}'
SERVICE         IMAGE                       STATUS          PORTS
alertmanager    prom/alertmanager:v0.27.0   Up 27 seconds   0.0.0.0:9093->9093/tcp, [::]:9093->9093/tcp
app             s20mon-app                  Up 27 seconds   0.0.0.0:8000->8000/tcp, [::]:8000->8000/tcp
grafana         grafana/grafana:12.1.1      Up 26 seconds   0.0.0.0:3000->3000/tcp, [::]:3000->3000/tcp
node-exporter   prom/node-exporter:v1.8.2   Up 27 seconds   9100/tcp
prometheus      prom/prometheus:v3.5.0      Up 27 seconds   0.0.0.0:9090->9090/tcp, [::]:9090->9090/tcp
```

### 1. Metrics

_Ask Prometheus with PromQL: `up` is 1 for every target it can scrape successfully (the second number of each pair)._

```console
$ curl -s localhost:9090/api/v1/query --data-urlencode 'query=up' | grep -o '"job":"[a-z-]*"\|"value":\[[^]]*\]' | paste - -
"job":"prometheus"	"value":[1792336801.956,"1"]
"job":"demo-app"	"value":[1792336801.956,"1"]
"job":"node"	"value":[1792336801.956,"1"]
```

![Prometheus targets](screenshots/prometheus-targets.png)

_Send some traffic to the app and read the raw metrics it exposes: a request counter per path and status, the health gauge, and the process memory and CPU time._

```console
$ for i in $(seq 1 25); do curl -s -o /dev/null localhost:8000/; done; for i in 1 2 3 4 5; do curl -s -o /dev/null "localhost:8000/work?ms=150"; done; curl -s localhost:8000/ ; echo; curl -s localhost:8000/metrics | grep -E "^app_http_requests_total|^app_healthy|^process_resident_memory_bytes|^process_cpu_seconds_total"
{"message":"Hello from the monitoring demo"}

process_resident_memory_bytes 3.3869824e+07
process_cpu_seconds_total 1.23
app_http_requests_total{path="/",status="200"} 26.0
app_http_requests_total{path="/work",status="200"} 5.0
app_healthy 1.0
```

### 2. Logs

The app writes one JSON object per request to stdout, which the container runtime stores.

_The last log lines (structured JSON: easy to filter and parse)._

```console
$ docker logs s20mon-app --tail 6
{"time": "2026-10-18T15:20:08", "level": "INFO", "message": "GET /work -> 200"}
{"time": "2026-10-18T15:20:08", "level": "INFO", "message": "GET /work -> 200"}
{"time": "2026-10-18T15:20:09", "level": "INFO", "message": "GET /work -> 200"}
{"time": "2026-10-18T15:20:09", "level": "INFO", "message": "GET /work -> 200"}
{"time": "2026-10-18T15:20:09", "level": "INFO", "message": "GET /work -> 200"}
{"time": "2026-10-18T15:20:09", "level": "INFO", "message": "GET / -> 200"}
```

_I trigger three failing requests and filter the log for errors. In a real cluster the same job is done centrally by Loki or an ELK/EFK stack (see the observability notes), and `kubectl logs` does it per Pod._

```console
$ for i in 1 2 3; do curl -s -o /dev/null localhost:8000/error; done; docker logs s20mon-app 2>&1 | grep "\"ERROR\"" | tail -n 3; echo "total INFO lines: $(docker logs s20mon-app 2>&1 | grep -c "\"INFO\"")  total ERROR lines: $(docker logs s20mon-app 2>&1 | grep -c "\"ERROR\"")"
{"time": "2026-10-18T15:20:10", "level": "ERROR", "message": "GET /error -> 500"}
{"time": "2026-10-18T15:20:10", "level": "ERROR", "message": "GET /error -> 500"}
{"time": "2026-10-18T15:20:10", "level": "ERROR", "message": "GET /error -> 500"}
total INFO lines: 31  total ERROR lines: 3
```

### 3. Application health

_The health endpoint: 200 means the application says it is fine._

```console
$ curl -s -w "  -> HTTP %{http_code}\n" localhost:8000/healthz; docker inspect -f "container state: {{.State.Status}}" s20mon-app
{"status":"ok"}
  -> HTTP 200
container state: running
```

### 4. The Grafana dashboard (healthy baseline)

Panels: app health, scrape target status, firing alerts, requests per second by status, error rate, latency p95, CPU utilisation, memory utilisation and the app's own memory.

![Grafana dashboard at baseline](screenshots/grafana-baseline.png)

### 5. Alerts

I now cause four different problems and watch the matching alert go from *pending* to *firing* in Prometheus and Alertmanager.

**Alert 1: high error rate** (half of the requests fail for about 40 seconds)

_Prometheus reports the alert HighErrorRate in state `firing` (it had to hold for 10 s first)._

```console
$ for i in $(seq 1 40); do curl -s -o /dev/null localhost:8000/error; curl -s -o /dev/null localhost:8000/; sleep 1; done; curl -s localhost:9090/api/v1/alerts | grep -o '"alertname":"[A-Za-z]*"\|"state":"[a-z]*"' | paste - -
"alertname":"HighErrorRate"	"state":"firing"
```

_The alert reached Alertmanager (state `active`)._

```console
$ curl -s localhost:9093/api/v2/alerts | grep -o '"alertname":"[A-Za-z]*"\|"state":"[a-z]*"'
"state":"active"
"alertname":"HighErrorRate"
```

![Prometheus alert page while HighErrorRate fires](screenshots/prometheus-alerts-error-rate.png)

**Alert 2: application health** (the app starts failing its health check)

_The health endpoint now returns 503, `app_healthy` becomes 0 and the AppUnhealthy alert fires._

```console
$ curl -s localhost:8000/break; echo; curl -s -w "  -> HTTP %{http_code}\n" localhost:8000/healthz; sleep 22; curl -s localhost:9090/api/v1/alerts | grep -o "\"alertname\":\"[A-Za-z]*\"\|\"state\":\"[a-z]*\"" | paste - -
{"status":"now unhealthy"}

{"status":"unhealthy"}
  -> HTTP 503
"alertname":"AppUnhealthy"	"state":"firing"
"alertname":"HighErrorRate"	"state":"firing"
```

_Repair it._

```console
$ curl -s localhost:8000/fix; echo; curl -s -w "  -> HTTP %{http_code}\n" localhost:8000/healthz
{"status":"healthy again"}

{"status":"ok"}
  -> HTTP 200
```

**Alert 3: CPU utilisation** (a stress container uses six CPU cores for about a minute)

_Start the CPU stress container._

```console
$ docker run -d --rm --name s20-stress --cpus=6 alpine sh -c "for i in 1 2 3 4 5 6; do (while :; do :; done) & done; sleep 70"
1f69075a193968fdd935964f7ef324b33d0641f7775ff56082cf955f51db8ffe
```

_**CPU utilisation** of the host (the second number, in percent) is well above the 40% threshold and HighCPUUsage is firing._

```console
$ curl -s localhost:9090/api/v1/query --data-urlencode 'query=100 - (avg(rate(node_cpu_seconds_total{mode="idle"}[30s])) * 100)' | grep -o '"value":\[[^]]*\]'; curl -s localhost:9090/api/v1/alerts | grep -o '"alertname":"[A-Za-z]*"\|"state":"[a-z]*"' | paste - -
"value":[1792336938.598,"54.6278886799742"]
"alertname":"HighCPUUsage"	"state":"firing"
```

The dashboard during the incident: the CPU panel shows the jump, the "firing alerts" tile is red, and the error rate and health panels show the earlier problems:

![Grafana dashboard during the incident](screenshots/grafana-incident.png)

_**Memory utilisation:** the host memory in percent (below the 85% threshold, so no memory alert), and the memory of the app process in MiB._

```console
$ curl -s localhost:9090/api/v1/query --data-urlencode 'query=(1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) * 100' | grep -o '"value":\[[^]]*\]'; curl -s localhost:9090/api/v1/query --data-urlencode 'query=process_resident_memory_bytes{job="demo-app"}/1024/1024' | grep -o '"value":\[[^]]*\]'
"value":[1792336947.189,"20.663285020404953"]
"value":[1792336947.317,"32.42578125"]
```

**Alert 4: the application disappears** (I stop its container)

_Prometheus can no longer scrape the target (`up` = 0), so **AppDown** fires. The earlier alerts had already resolved by then, so only AppDown is listed._

```console
$ docker stop s20mon-app && sleep 28 && curl -s localhost:9090/api/v1/alerts | grep -o '"alertname":"[A-Za-z]*"\|"state":"[a-z]*"' | paste - -
s20mon-app
"alertname":"AppDown"	"state":"firing"
```

_Start it again: the target is back (`up` = 1) and the alert resolves._

```console
$ docker start s20mon-app && sleep 20 && curl -s localhost:9090/api/v1/query --data-urlencode "query=up{job=\"demo-app\"}" | grep -o "\"value\":\[[^]]*\]"
s20mon-app
"value":[1792337037.114,"1"]
```

![Prometheus alert page with AppDown firing](screenshots/prometheus-alerts-app-down.png)

_After the problems are fixed and the stress container has ended, no alert is firing._

```console
$ curl -s localhost:9090/api/v1/alerts | grep -o '"alertname":"[A-Za-z]*"\|"state":"[a-z]*"' | paste - - ; echo '(empty = nothing is firing any more)'
(empty = nothing is firing any more)
```

### What each alert taught

| Alert | Metric used | Cause I created | Typical real-world meaning |
|---|---|---|---|
| HighErrorRate | `app_http_requests_total{status=~"5.."}` | `/error` requests | a bad release or a broken dependency |
| AppUnhealthy | `app_healthy` | `/break` | the app knows it cannot work (database lost...) |
| HighCPUUsage | `node_cpu_seconds_total` | stress container | load spike, runaway process, time to scale |
| AppDown | `up` | stopped container | crash, node failure, wrong network config |

Memory follows the same pattern (`HighMemoryUsage` on `node_memory_MemAvailable_bytes`); I checked its current value above.

_Cleanup: stop and remove the whole demo stack._

```console
$ cd monitoring-demo && docker compose down -v --rmi local 2>&1 | tail -n 8
 Container s20mon-alertmanager Removed 
 Container s20mon-app Stopped 
 Container s20mon-app Removing 
 Container s20mon-app Removed 
 Image s20mon-app:latest Removing 
 Network s20mon_default Removing 
 Image s20mon-app:latest Removed 
 Network s20mon_default Removed 
```

---

## Task 2: Observability

**Monitoring** answers questions I thought of in advance ("is the CPU above 40%?", "is the app down?"). **Observability** is the ability to answer questions I did **not** think of in advance by looking at what the system emits. It rests on three pillars:

| Pillar | What it is | Answers | Typical tools | In my demo (Task 1) |
|---|---|---|---|---|
| **Metrics** | numbers sampled over time (counters, gauges, histograms) | *how much / how fast / how often?* | Prometheus, Grafana, Datadog | Prometheus scraped `/metrics`; Grafana dashboard; alert rules |
| **Logs** | timestamped records of events, ideally structured (JSON) | *what exactly happened?* | Loki, ELK/EFK, Fluent Bit, `kubectl logs` | the app's JSON logs, filtered with `docker logs` and `grep` |
| **Traces** | the path of one request across services, with the time spent in each | *where did this request spend its time, and where did it fail?* | OpenTelemetry, Jaeger, Tempo | **not implemented** in the demo (a single-service app has nothing to trace across); explained in the notes |

**Why observability is required:** modern systems are distributed and change often, so failures are novel. Good signals reduce the time to detect and to repair (MTTD, MTTR), let teams define SLOs and alert on user impact instead of guessing, and make the difference between "the server is up" and "users can check out".

**Kubernetes observability** is built from the same pieces: the metrics-server (`kubectl top`, the HPA), `kubectl logs` and `kubectl get events` for logs and cluster events, kube-state-metrics and node-exporter for object and node metrics, and Prometheus with Grafana on top. I used these for real in earlier sessions: `kubectl top` and the HPA metrics in [Session 13](../session13-storage-hpa-probes/README.md), events, logs and `describe` as the main diagnostic tools in [Session 14](../session14-kubernetes-troubleshooting/README.md), and Prometheus scraping the real StockPilot backend in the [final project](../session21-final-devops-project/final-devops-project/README.md#11-monitoring).

The full written notes (tools per pillar, golden signals, RED and USE methods, example PromQL and an alert rule) are in [observability/README.md](observability/README.md).

---

_(the Argo CD demonstration is recorded after the repository has been pushed to GitHub)_
