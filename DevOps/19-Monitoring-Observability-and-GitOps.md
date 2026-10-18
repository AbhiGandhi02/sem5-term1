# Monitoring, Observability & GitOps (Prometheus, Grafana, Argo CD)

> **DevOps – Lecture 19 notes.** Anything marked **➕ Extra** was not covered in class. It's there because it fills a gap or comes up often in labs and interviews.
>
> **Where this fits:** CI/CD ([L15](15-CICD-with-GitHub-Actions.md), [L16](16-End-to-End-Deployment-Project.md)) gets the app **into** production. This lecture covers what happens **after**: **watching** it (Prometheus + Grafana) and **keeping the cluster in sync with Git** (GitOps with Argo CD). Builds on [Docker Compose](05-Docker.md#11-docker-compose-multi-container-apps), [Docker networking](07-Docker-Networking-and-Volumes.md#3-the-four-network-types) and [Kubernetes Deployments](09-Kubernetes-Objects-Services-Deployments.md#6-deployments).

---

## Table of Contents
1. [The Big Picture: Monitoring vs. Observability](#1-the-big-picture-monitoring-vs-observability)
2. [Prometheus: How It Works](#2-prometheus-how-it-works)
3. [Metrics & Metric Types](#3-metrics--metric-types)
4. [Hands-on: Prometheus + Grafana with Docker Compose](#4-hands-on-prometheus--grafana-with-docker-compose)
5. [PromQL: Querying Metrics](#5-promql-querying-metrics)
6. [Grafana: Data Sources, Dashboards & Panels](#6-grafana-data-sources-dashboards--panels)
7. [➕ Instrumenting Your Own App](#7--instrumenting-your-own-app)
8. [➕ Alerting](#8--alerting)
9. [GitOps: Git as the Source of Truth](#9-gitops-git-as-the-source-of-truth)
10. [Argo CD: Architecture & Concepts](#10-argo-cd-architecture--concepts)
11. [Hands-on: Install Argo CD & Sync an App](#11-hands-on-install-argo-cd--sync-an-app)
12. [➕ Putting It Together: CI + GitOps + Monitoring](#12--putting-it-together-ci--gitops--monitoring)
13. [Quick Revision Cheat Sheet](#13-quick-revision-cheat-sheet)
14. [Likely Exam / Interview Questions](#14-likely-exam--interview-questions)

---

## 1. The Big Picture: Monitoring vs. Observability

| | **Monitoring** | **Observability** |
|-|----------------|-------------------|
| Question | "**Is** something wrong?" | "**Why** is it wrong?" |
| Approach | Watch **known** metrics, alert on thresholds (CPU > 90%) | Collect enough data to investigate **unknown** problems |
| Example | "API error rate is 5%" 🚨 | "Errors come only from `/api/calculate` on Pod `web-2`, after the 14:05 deploy" |

> 💡 **Simple words mein:** monitoring = car ka dashboard (speed, petrol, warning light). Observability = mechanic ka poora diagnostic kit, jo batata hai warning light **kyun** jal rahi hai.

### ➕ The three pillars of observability
| Pillar | What | Tool examples |
|--------|------|---------------|
| **Metrics** | Numbers over time (CPU %, requests/s, latency) | **Prometheus** + **Grafana** (this lecture) |
| **Logs** | Text events ("ERROR: DB timeout") | `kubectl logs` ([L13](13-Kubernetes-Troubleshooting-and-Helm.md#4-kubectl-logs-what-did-the-app-say)), Loki, ELK stack |
| **Traces** | One request's journey across services | Jaeger, Tempo, OpenTelemetry |

### ➕ What to watch: the four golden signals (Google SRE)
| Signal | Meaning | Example |
|--------|---------|---------|
| **Latency** | How long requests take | p95 response time = 180 ms |
| **Traffic** | How much demand | 250 requests/s |
| **Errors** | How many fail | 0.5% HTTP 5xx |
| **Saturation** | How "full" the system is | CPU 85%, memory 90%, disk 95% |

---

## 2. Prometheus: How It Works

**Prometheus** is an open-source **monitoring system and time-series database**. It **collects metrics**, stores them with timestamps, lets you **query** them (PromQL), and evaluates **alert rules**.

```
                        PULL (scrape) every 15 s
  ┌──────────────┐   GET /metrics   ┌─────────────────────────────┐   PromQL   ┌──────────┐
  │ your app     │ ◀─────────────── │        PROMETHEUS           │ ◀───────── │ GRAFANA  │
  │ :5001/metrics│                  │  ┌───────────┐ ┌──────────┐ │            │dashboards│
  └──────────────┘                  │  │ scraper   │ │ TSDB     │ │            └──────────┘
  ┌──────────────┐   GET /metrics   │  │ (targets) │ │(storage) │ │
  │ node-exporter│ ◀─────────────── │  └───────────┘ └──────────┘ │ alerts ──▶ ┌──────────────┐
  │ :9100 (host) │                  │  rule evaluation            │ ─────────▶ │ Alertmanager │──▶ Slack/email
  └──────────────┘                  └─────────────────────────────┘            └──────────────┘
```

| Concept | Meaning |
|---------|---------|
| **Pull model** | Prometheus **visits** each target and fetches `/metrics` over HTTP. Apps don't send data. |
| **Target** | One endpoint to scrape, e.g. `node-exporter:9100` |
| **Job** | A group of targets of the same kind, e.g. `job="node"` |
| **Exporter** | A helper that exposes metrics for something that can't do it itself: **node-exporter** (host CPU/RAM/disk), cAdvisor (containers), mysqld-exporter … |
| **TSDB** | Time-series database: stores `(metric, labels, timestamp, value)` on local disk |
| **Alertmanager** ➕ | Separate component that groups and routes alerts to Slack, email, PagerDuty |

> ⚠️ **Precision point:** the class notes say Prometheus offers "alerts and visualizations". Prometheus **evaluates** alert rules, but **sending** notifications is **Alertmanager's** job. Its built-in Graph page is basic, so real **dashboards are Grafana's** job.
>
> 💡 **Simple words mein:** Prometheus = har 15 second mein har dukaan pe jaake register (metrics) padhne wala inspector. Grafana = us data se sundar charts banane wala designer.

---

## 3. Metrics & Metric Types

**What a metric looks like** (open `http://localhost:9100/metrics`):
```
# HELP node_cpu_seconds_total Seconds the CPUs spent in each mode.
# TYPE node_cpu_seconds_total counter
node_cpu_seconds_total{cpu="0",mode="idle"} 12345.67
node_cpu_seconds_total{cpu="0",mode="user"} 234.5
```
`metric_name{label="value", ...} value`. **Labels** let you filter and group (by CPU, by Pod, by endpoint), just like K8s labels ([L9 §4](09-Kubernetes-Objects-Services-Deployments.md#4-labels--selectors)).

| ➕ Type | Behaviour | Example | Query it with |
|--------|-----------|---------|---------------|
| **Counter** | Only goes **up** (resets on restart) | `http_requests_total` | `rate()`: per-second speed |
| **Gauge** | Goes **up and down** | `node_memory_MemAvailable_bytes`, temperature | Use directly, `avg`, `max` |
| **Histogram** | Counts values in **buckets** | `http_request_duration_seconds_bucket` | `histogram_quantile()` for p95/p99 |
| **Summary** | Pre-calculated quantiles in the app | `..._duration_seconds{quantile="0.95"}` | Use directly |

---

## 4. Hands-on: Prometheus + Grafana with Docker Compose

As in class, both run as **containers managed by Docker Compose** ([L5 §11](05-Docker.md#11-docker-compose-multi-container-apps)).

```
monitoring/
├── docker-compose.yml
└── prometheus/
    └── prometheus.yml        # mounted INTO the container
```

### `prometheus/prometheus.yml`
```yaml
global:
  scrape_interval: 15s              # how often to pull metrics

scrape_configs:
  - job_name: prometheus            # Prometheus monitors itself
    static_configs:
      - targets: ["localhost:9090"]

  - job_name: node                  # host metrics from node-exporter
    static_configs:
      - targets: ["node-exporter:9100"]   # container NAME as hostname (Compose DNS)

  # ➕ your app from L16, once it exposes /metrics (§7)
  # - job_name: python-web
  #   static_configs:
  #     - targets: ["python-web:5001"]
```
> **Note:** the class notes call it `prometheus.yaml`. Prometheus's default config file is **`prometheus.yml`** at `/etc/prometheus/prometheus.yml` inside the container. Any name works if you mount it to that path.

### `docker-compose.yml`
```yaml
services:
  prometheus:
    image: prom/prometheus:v2.53.0
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro   # config (bind mount, read-only)
      - prometheus-data:/prometheus                                      # ➕ keep metrics across restarts

  node-exporter:
    image: prom/node-exporter:v1.8.1
    ports:
      - "9100:9100"

  grafana:
    image: grafana/grafana:11.1.0
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin     # ➕ lab only: change it in real setups
    volumes:
      - grafana-data:/var/lib/grafana        # ➕ keep dashboards across restarts
    depends_on:
      - prometheus

volumes:
  prometheus-data:
  grafana-data:
```
> ⚠️ **Correction:** the class notes say Grafana runs "with a built-in Prometheus dependency". Grafana **doesn't include** Prometheus. `depends_on` only makes Compose **start Prometheus first**. They're two separate containers, and you **connect** them by adding Prometheus as a **data source** in Grafana (§6).

### Run it
```bash
cd monitoring
docker compose up -d
docker compose ps                              # 3 containers Up
```
| URL | What |
|-----|------|
| `http://localhost:9090` | Prometheus UI → **Status → Targets**: all should be **UP** |
| `http://localhost:9100/metrics` | Raw node-exporter metrics |
| `http://localhost:3000` | Grafana (login `admin` / `admin`) |

**First query (from class):** in Prometheus, type **`up`** → Execute:
```
up{instance="localhost:9090", job="prometheus"}     1
up{instance="node-exporter:9100", job="node"}       1     ← 1 = scrape worked, 0 = target DOWN
```

> ➕ On Docker Desktop (Mac/Windows), node-exporter reports the **hidden Linux VM's** CPU and memory, not your laptop's. On a Linux server it reports the real host.
>
> **Volumes recap** ([L7 §8](07-Docker-Networking-and-Volumes.md#8-named-volumes-vs-bind-mounts)): the config is a **bind mount** (you edit it on the host), and the data is in **named volumes** (it survives `docker compose down`; `down -v` deletes it).

---

## 5. PromQL: Querying Metrics

**PromQL** (Prometheus Query Language) is the language you use in both Prometheus and Grafana.

| Query | Answers |
|-------|---------|
| `up` | Which targets are up (1) or down (0)? |
| `up == 0` | ➕ Only the **down** targets |
| `node_memory_MemAvailable_bytes / 1024 / 1024` | Free memory in MB |
| `100 * (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)` | **Memory used %** |
| `rate(node_cpu_seconds_total{mode!="idle"}[5m])` | CPU seconds used per second, over the last 5 min |
| `100 * (1 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])))` | **CPU used %** (all cores) |
| `rate(http_requests_total[1m])` | ➕ Requests per second (traffic) |
| `sum by (endpoint) (rate(http_requests_total[1m]))` | ➕ Requests/s **per endpoint** |
| `histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))` | ➕ **p95 latency** |

**Key ideas:**
- `{label="x"}` filters, `{label!="x"}` excludes, `{label=~"a|b"}` is a regex match.
- `[5m]` is a **range**: the last 5 minutes of samples. Needed by `rate()`.
- **`rate()` on counters, always.** The raw value of a counter (total since start) is rarely useful. Its **rate of increase** is.
- `sum`, `avg`, `max`, plus `by (label)`, group results like SQL `GROUP BY`.

> 💡 **Simple words mein:** counter = gaadi ka odometer (total km, sirf badhta hai). `rate()` = speedometer (abhi kitni tez chal rahi hai). Dashboard pe speed chahiye, total km nahi.

---

## 6. Grafana: Data Sources, Dashboards & Panels

### Step 1: add Prometheus as a data source
**Connections → Data sources → Add data source → Prometheus**
```
URL:  http://prometheus:9090        ← the Compose SERVICE NAME
```
→ **Save & test** → ✅ "Successfully queried the Prometheus API".

> ⚠️ **Classic mistake:** entering `http://localhost:9090`. Grafana runs **in its own container**, so `localhost` there means **the Grafana container itself**, not your laptop. Containers on the same Compose network reach each other **by service name** ([L7 §3.1](07-Docker-Networking-and-Volumes.md#31-bridge-default)). That's why the class used `http://prometheus:9090`.

### Step 2: build a dashboard
**Dashboards → New → Add visualization → choose Prometheus →** type a PromQL query → choose a visualization:

| Visualization | Best for | Example query |
|---------------|----------|---------------|
| **Time series** (graph) | Trends over time | CPU % over the last hour |
| **Gauge** | One current value against a range | Memory used % (green → red) |
| **Stat** | A big single number | Targets up: `sum(up)` |
| **Bar chart / Bar gauge** | Comparing items | Requests per endpoint |
| **Table** | Exact values, many series | `up` for every target |

Set **units** (percent, bytes, seconds) and **thresholds** (e.g. red above 80%) in the panel options → **Save dashboard**.

### ➕ Shortcuts
- **Import a ready-made dashboard:** Dashboards → New → **Import** → ID **`1860`** ("Node Exporter Full") → select the Prometheus data source. You instantly get dozens of host panels.
- **Provisioning:** data sources and dashboards can be defined as **YAML/JSON files** mounted into Grafana, so nothing needs clicking. That's the IaC idea ([L17](17-Terraform-IaC.md#1-the-big-picture-what-is-iac)) applied to dashboards.

---

## 7. ➕ Instrumenting Your Own App

node-exporter shows **host** health. To see **app** metrics (requests, errors, latency), the app must expose `/metrics`. For the Flask dashboard from [L16](16-End-to-End-Deployment-Project.md#2-the-application):

```python
# pip install prometheus-client
from prometheus_client import Counter, Histogram, generate_latest, CONTENT_TYPE_LATEST
from flask import Flask, Response, request
import time

app = Flask(__name__)

REQUESTS = Counter("http_requests_total", "Total HTTP requests", ["endpoint", "status"])
LATENCY  = Histogram("http_request_duration_seconds", "Request latency", ["endpoint"])

@app.before_request
def start_timer():
    request._start = time.time()

@app.after_request
def record(response):
    LATENCY.labels(request.path).observe(time.time() - request._start)
    REQUESTS.labels(request.path, response.status_code).inc()
    return response

@app.route("/metrics")
def metrics():
    return Response(generate_latest(), mimetype=CONTENT_TYPE_LATEST)
```
Then uncomment the `python-web` job in `prometheus.yml`. Now you can graph **traffic**, **errors** (`status=~"5.."`) and **latency** (p95): three of the four golden signals.

> In Kubernetes, the standard bundle is the **kube-prometheus-stack** Helm chart ([L13 Helm](13-Kubernetes-Troubleshooting-and-Helm.md#10-helm-why-a-package-manager)): Prometheus + Alertmanager + Grafana + node-exporter + ready-made K8s dashboards in one `helm install`.

---

## 8. ➕ Alerting

Dashboards need someone watching them. **Alerts** come to you. Add a rule file to Prometheus:

```yaml
# prometheus/alerts.yml   (reference it in prometheus.yml under rule_files:)
groups:
  - name: basic
    rules:
      - alert: TargetDown
        expr: up == 0
        for: 1m                          # must stay true for 1 minute (avoids flapping)
        labels:
          severity: critical
        annotations:
          summary: "{{ $labels.instance }} is down"

      - alert: HighMemory
        expr: 100 * (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes) > 90
        for: 5m
        labels:
          severity: warning
```
```
 Prometheus evaluates rules ──firing──▶ Alertmanager ──▶ Slack / email / PagerDuty
                                         (groups, de-duplicates, silences)
```
Grafana can also define alerts directly on panels.

---

## 9. GitOps: Git as the Source of Truth

**GitOps** is an **approach** (the class notes' "framework that approves…" means "approach") where the **desired state** of your apps and infrastructure is stored as **declarative files in Git**, and an **automated agent** keeps the real system **continuously in sync** with Git.

### The four GitOps principles (➕ OpenGitOps)
| Principle | Meaning |
|-----------|---------|
| **Declarative** | The system is described as *what* it should be (K8s YAML, Helm, Kustomize) |
| **Versioned & immutable** | Stored in Git, so there's full history, review through PRs, and every state can be restored |
| **Pulled automatically** | An agent **inside** the cluster pulls changes. Nobody pushes `kubectl apply` from outside. |
| **Continuously reconciled** | The agent keeps comparing Git ↔ cluster and **fixes any difference** (drift) |

It's the same desired-state loop as Kubernetes controllers ([L8 §6](08-Kubernetes-Basics.md#6-controllers-node--replicaset)) and Terraform ([L17 §10](17-Terraform-IaC.md#10--state-how-terraform-remembers)), but the "desired state" lives **in Git**.

### Push-based CD vs. GitOps (pull-based)
```
 PUSH (classic CI/CD, L15–16)                    PULL (GitOps)
 CI pipeline ──kubectl apply / helm upgrade──▶   CI pipeline ──commit new image tag──▶ Git repo
      cluster                                                                            ▲
 (CI needs cluster admin credentials ⚠️)                     Argo CD (inside cluster) ──pulls┘
                                                             └──applies──▶ cluster
```

| | Push-based CD | **GitOps (pull)** |
|-|---------------|-------------------|
| Who changes the cluster | The CI pipeline, from outside | An agent **inside** the cluster |
| Cluster credentials in CI | **Required** (a risk) | **Not needed** |
| Someone runs `kubectl edit` by hand | Change stays (drift goes unnoticed) | **Detected and reverted** (self-heal) |
| Rollback | Re-run an old pipeline | **`git revert`** |
| Audit trail | Pipeline logs | **Git history**: who changed what, when, and why (PR) |

> 💡 **Simple words mein:** GitOps mein Git hi "boss" hai. Cluster mein jo bhi ho, Argo CD dekhta rehta hai ki Git mein kya likha hai, aur cluster ko waisa hi bana deta hai. Kisi ne haath se kuch badla? Argo CD wapas Git jaisa kar dega.

---

## 10. Argo CD: Architecture & Concepts

**Argo CD** is a **GitOps continuous-delivery tool for Kubernetes**. It runs **inside** the cluster, watches Git repos, and syncs K8s resources to match them.

```
 Git repo (k8s/deployment.yaml, service.yaml)
        ▲ polls every ~3 min (or webhook)
        │
 ┌──────┴──────────── namespace: argocd ─────────────────┐
 │ repo-server         → clones repo, renders YAML/Helm   │
 │ application-controller → compares Git vs cluster,      │
 │                        syncs, self-heals               │
 │ argocd-server       → web UI + API + CLI               │
 └──────────────────────────┬─────────────────────────────┘
                            │ kubectl-like apply
                            ▼
                namespace: my-app  (Deployment, Service …)
```

| Concept | Meaning |
|---------|---------|
| **Application** | An Argo CD object (a CRD): "this **Git path** → this **cluster/namespace**" |
| **Sync status** | **Synced** (cluster = Git) or **OutOfSync** (they differ) |
| **Health status** | **Healthy**, **Progressing** (rolling out), **Degraded** (e.g. CrashLoopBackOff), Missing |
| **Sync** | Apply Git's state to the cluster (manual button, or automatic) |
| **Automated sync** | Sync by itself whenever Git changes |
| **Prune** | Delete cluster resources that were **removed from Git** |
| **Self-heal** | Undo **manual changes** made in the cluster (drift) |

**Namespaces (from class):** Argo CD itself lives in the **`argocd`** namespace. Your app goes into its **own target namespace** (e.g. `my-app`), which keeps the tool and the workloads separate ([L9 §3](09-Kubernetes-Objects-Services-Deployments.md#3-namespaces)).

---

## 11. Hands-on: Install Argo CD & Sync an App

### Step 1: put the app manifests in Git
A GitHub repo (e.g. `gitops-demo`) with the Deployment + Service from [L16 §5](16-End-to-End-Deployment-Project.md#5-stage-3-run-it-on-kubernetes-minikube) (or plain nginx):
```
gitops-demo/
└── k8s/
    ├── deployment.yaml      # replicas: 2, image: nginx:1.27
    └── service.yaml
```

### Step 2: install Argo CD with kubectl
```bash
minikube start
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl get pods -n argocd -w            # wait until all are Running
```

### Step 3: open the UI
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443      # keep this terminal open
# initial admin password (base64, like any Secret: L11 §5)
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```
Open **https://localhost:8080** (accept the self-signed certificate) → user **`admin`** + that password.

### Step 4: create the Application (declaratively, the GitOps way)
**`argocd-app.yaml`**
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gitops-demo
  namespace: argocd                      # Application objects live in Argo CD's namespace
spec:
  project: default
  source:
    repoURL: https://github.com/<your-user>/gitops-demo.git
    targetRevision: main                 # branch / tag / commit to follow
    path: k8s                            # folder with the manifests
  destination:
    server: https://kubernetes.default.svc   # this same cluster
    namespace: my-app                    # target namespace for the app
  syncPolicy:
    automated:
      prune: true                        # delete resources removed from Git
      selfHeal: true                     # undo manual changes in the cluster
    syncOptions:
      - CreateNamespace=true             # create "my-app" if it doesn't exist
```
```bash
kubectl apply -f argocd-app.yaml
kubectl get applications -n argocd       # SYNC STATUS: Synced   HEALTH STATUS: Healthy
kubectl get pods -n my-app               # 2 Pods, created by Argo CD, not by you
```
(The same can be done by clicking **+ NEW APP** in the UI.)

### Step 5: the demo from class: change replicas **in Git**
```bash
# in the gitops-demo repo: change replicas: 2 → 4
git commit -am "Scale to 4 replicas" && git push
```
Within ~3 minutes (Argo CD's default polling; ➕ instant with a Git webhook, or click **Refresh**), the app goes **OutOfSync → Syncing → Synced**:
```bash
kubectl get pods -n my-app               # now 4 Pods ✅, and nobody ran kubectl
```

### Step 6: ➕ watch self-heal
```bash
kubectl scale deploy <name> -n my-app --replicas=1     # a "manual hotfix" outside Git
kubectl get pods -n my-app -w                          # Argo CD scales it back to 4
```
**In GitOps, every change goes through Git.** A manual change is drift, and Argo CD reverts it.

### ➕ Rollback the GitOps way
```bash
git revert HEAD && git push              # Argo CD syncs the previous state
```

### ➕ The `argocd` CLI
```bash
argocd login localhost:8080 --username admin --insecure
argocd app list
argocd app get gitops-demo
argocd app sync gitops-demo              # manual sync
argocd app history gitops-demo
```

---

## 12. ➕ Putting It Together: CI + GitOps + Monitoring

```
 developer ──git push (app code)──▶ CI (GitHub Actions, L15–16)
                                    test → SAST → SCA → build → Trivy → push image :sha
                                    └──▶ commit "image: user/app:<sha>" to the GITOPS repo
                                                   │
                                                   ▼
                                   Argo CD (in cluster) sees the new commit → sync
                                                   │
                                                   ▼
                              Kubernetes rolling update (L10) → new Pods
                                                   │
                                                   ▼
                     Prometheus scrapes /metrics → Grafana dashboards → Alertmanager alerts
                                    (bad release? → git revert → Argo CD rolls back)
```
- **CI** proves the code is good and produces an image.
- **GitOps (Argo CD)** deploys exactly what Git says.
- **Monitoring** tells you whether the release is healthy in production.

---

## 13. Quick Revision Cheat Sheet

```
MONITORING    IS something wrong (known metrics, thresholds) · OBSERVABILITY = WHY (explore unknowns)
3 PILLARS ➕  metrics (Prometheus) · logs (Loki/ELK) · traces (Jaeger/OTel)
GOLDEN ➕     latency · traffic · errors · saturation
PROMETHEUS    metrics + time-series DB + PromQL + alert rules · PULL: scrapes GET /metrics every 15s
              target · job · exporter (node-exporter :9100) · Alertmanager SENDS alerts · UI :9090
METRIC TYPES  counter (only up → rate()) · gauge (up/down) · histogram (buckets → histogram_quantile) · summary
CONFIG        prometheus.yml → /etc/prometheus/prometheus.yml (bind mount) · global.scrape_interval
              scrape_configs: job_name + static_configs.targets ["node-exporter:9100"]
COMPOSE       prometheus (9090) · node-exporter (9100) · grafana (3000) · depends_on = start ORDER only
              named volumes for data · docker compose up -d · Status → Targets = UP
PROMQL        up · up == 0 · rate(counter[5m]) · sum by (label) (...) · histogram_quantile(0.95, ...)
              {l="x"} {l!="x"} {l=~"a|b"} · CPU% / memory% formulas
GRAFANA       :3000 admin/admin · data source URL http://prometheus:9090 (service name, NOT localhost)
              dashboard → panel → query → time series / gauge / stat / bar / table · import ID 1860 ➕
GITOPS        Git = source of truth · declarative · versioned · PULLED by agent · continuously reconciled
              no cluster creds in CI · rollback = git revert · drift auto-corrected
ARGO CD       runs in namespace argocd · repo-server · application-controller · argocd-server (UI/API)
              install: kubectl create ns argocd; kubectl apply -n argocd -f .../stable/manifests/install.yaml
              UI: port-forward svc/argocd-server 8080:443 · password: secret argocd-initial-admin-secret
APPLICATION   source (repoURL, targetRevision, path) → destination (server, namespace)
              syncPolicy.automated { prune, selfHeal } · CreateNamespace=true
STATUS        Synced / OutOfSync · Healthy / Progressing / Degraded · polls Git ~3 min (webhook = instant)
DEMO          change replicas in Git → push → Argo CD syncs · kubectl scale by hand → self-heal reverts
```

---

## 14. Likely Exam / Interview Questions

1. **Monitoring vs. observability?** Monitoring tells you that something is wrong, using known metrics and thresholds. Observability lets you work out why, from rich data (metrics, logs, traces).
2. **What are the three pillars of observability?** Metrics, logs, traces.
3. **What are the four golden signals?** Latency, traffic, errors, saturation.
4. **Does Prometheus push or pull metrics? What is scraped?** It pulls: it periodically sends HTTP GET requests to each target's `/metrics` endpoint.
5. **What is an exporter? Give an example.** A helper process that exposes metrics for a system that can't do it itself, e.g. node-exporter for host CPU, memory and disk.
6. **Counter vs. gauge? Which function do you use with counters?** A counter only increases. A gauge goes up and down. Use `rate()` on counters.
7. **What does the PromQL query `up` return?** 1 for each target scraped successfully and 0 for targets that are down.
8. **Write a PromQL query for memory used %.** `100 * (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)`.
9. **Prometheus vs. Grafana?** Prometheus collects, stores and queries metrics. Grafana visualizes data from Prometheus and other sources in dashboards.
10. **In Docker Compose, why is Grafana's data source URL `http://prometheus:9090` and not `localhost:9090`?** Each container has its own `localhost`. Containers on the same Compose network reach each other by service name.
11. **What does `depends_on` do in Compose? Does it bundle Prometheus into Grafana?** It only controls start order. They're separate containers, connected through a data source.
12. **Who sends alert notifications in the Prometheus ecosystem?** Alertmanager. Prometheus only evaluates the alert rules.
13. **What is GitOps, and what are its principles?** Git holds the declarative desired state, an agent pulls it automatically, and it continuously reconciles the cluster to match.
14. **Push-based CD vs. GitOps?** In push-based CD, CI applies changes to the cluster and needs its credentials. In GitOps, an in-cluster agent pulls from Git, giving audit through Git history, rollback with `git revert`, and drift correction.
15. **What is Argo CD, and where does it run?** A GitOps CD tool for Kubernetes. It runs inside the cluster, usually in the `argocd` namespace.
16. **What does an Argo CD Application define?** A source (repo URL, revision, path) and a destination (cluster and namespace), plus a sync policy.
17. **Synced/OutOfSync vs. Healthy/Degraded?** Whether the cluster matches Git, vs. whether the deployed resources are actually working.
18. **What do `prune` and `selfHeal` do?** Prune deletes resources that were removed from Git. Self-heal reverts manual changes made in the cluster.
19. **You change `replicas` from 2 to 4 in Git. What happens, and how long does it take?** Argo CD detects the new commit (polling every ~3 min, or instantly with a webhook), syncs, and the Deployment scales to 4.
20. **With auto-sync and self-heal on, an engineer runs `kubectl scale --replicas=1`. What happens?** Argo CD sees the drift and scales back to the value in Git. Changes must be made through Git.
21. **How do you get Argo CD's initial admin password?** Decode the `argocd-initial-admin-secret` Secret in the `argocd` namespace (base64).
