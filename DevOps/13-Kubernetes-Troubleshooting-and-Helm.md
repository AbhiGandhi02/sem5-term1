# Kubernetes Part 6: Troubleshooting & Helm

> **DevOps – Lecture 13 notes.** Anything marked **➕ Extra** was not covered in class. It's there because it fills a gap or comes up often in labs and interviews.
>
> **Builds on:** every Kubernetes lecture so far, [8](08-Kubernetes-Basics.md) → [12](12-Kubernetes-Volumes-and-Autoscaling.md). The errors here are the ones you've already met in labs. This lecture puts the **debugging method** in one place and introduces **Helm** for packaging apps.
>
> **Practice:** [Kubernetes MCQs](Kubernetes-MCQs.md).

---

## Table of Contents
1. [The Troubleshooting Mindset](#1-the-troubleshooting-mindset)
2. [`kubectl get`: What's the State?](#2-kubectl-get-whats-the-state)
3. [`kubectl describe`: Why Is It in That State?](#3-kubectl-describe-why-is-it-in-that-state)
4. [`kubectl logs`: What Did the App Say?](#4-kubectl-logs-what-did-the-app-say)
5. [`kubectl exec`: Look Inside](#5-kubectl-exec-look-inside)
6. [Events: The Cluster's Diary](#6-events-the-clusters-diary)
7. [Common Errors & How to Fix Them](#7-common-errors--how-to-fix-them)
8. [Troubleshooting Flowchart](#8-troubleshooting-flowchart)
9. [Hands-on: Break It & Fix It](#9-hands-on-break-it--fix-it)
10. [Helm: Why a Package Manager?](#10-helm-why-a-package-manager)
11. [Helm Key Terms](#11-helm-key-terms)
12. [Helm Basic Commands](#12-helm-basic-commands)
13. [Creating a Custom Chart](#13-creating-a-custom-chart)
14. [Templates & Values: How Helm Works](#14-templates--values-how-helm-works)
15. [Upgrade, Rollback & Environments](#15-upgrade-rollback--environments)
16. [Hands-on: `demo-chart` End to End](#16-hands-on-demo-chart-end-to-end)
17. [Quick Revision Cheat Sheet](#17-quick-revision-cheat-sheet)
18. [Likely Exam / Interview Questions](#18-likely-exam--interview-questions)

---

## 1. The Troubleshooting Mindset

Five commands answer five questions, **in this order**:

| # | Command | Question it answers |
|---|---------|---------------------|
| 1 | `kubectl get` | **What** is the state? (Running? Pending? CrashLoopBackOff?) |
| 2 | `kubectl describe` | **Why** is it in that state? (read the **Events**) |
| 3 | `kubectl logs` | **What did the app say** before failing? |
| 4 | `kubectl exec` | **What does it look like inside** the container? |
| 5 | `kubectl get events` | **What happened across the cluster** recently? |

> 💡 **Simple words mein:** doctor wala tareeka. `get` = bukhaar hai ya nahi (symptom). `describe` = report padho, kyun hai. `logs` = patient se poocho kya hua. `exec` = andar jaake check-up.

**Golden rule:** read the **STATUS** first. It tells you **where** to look:
```
 Pending            → scheduling problem   → describe (Events)
 ImagePullBackOff   → image problem        → describe (Events)
 CrashLoopBackOff   → app problem          → logs --previous
 Running, not working → network/config     → exec, Service endpoints
```

---

## 2. `kubectl get`: What's the State?

```bash
kubectl get pods                    # Pods in the current namespace (ALL phases, not only running)
kubectl get pods -w                 # watch: prints a line every time a status changes
kubectl get pods -o wide            # + Pod IP and NODE (is everything failing on one node?)
kubectl get pods -A                 # all namespaces
kubectl get pods --show-labels      # check labels against Service/Deployment selectors
kubectl get deploy,rs,svc,pods      # several types at once
kubectl get pod <pod> -o yaml       # ➕ the full stored spec + live status
```

**Read these columns:**
```
NAME                    READY   STATUS             RESTARTS      AGE
web-7d9f-abc12          1/1     Running            0             5m     ✅
web-7d9f-def34          0/1     CrashLoopBackOff   6 (40s ago)   5m     ❌ app keeps dying
api-5c8b-ghi56          0/1     ImagePullBackOff   0             2m     ❌ can't get the image
db-0                    0/1     Pending            0             10m    ❌ not scheduled
```
| Column | Red flag |
|--------|----------|
| `READY 0/1` | The container isn't ready (crashing, or its readiness probe is failing) |
| `RESTARTS` climbing | The container keeps exiting |
| `STATUS` ≠ Running/Completed | See [§7](#7-common-errors--how-to-fix-them) |

---

## 3. `kubectl describe`: Why Is It in That State?

```bash
kubectl describe pod <pod-name>
kubectl describe deploy <name>      # works on any object: node, svc, pvc, ingress, hpa …
```

**The parts that matter:**
```
Name:         api-5c8b-ghi56
Node:         minikube/192.168.49.2
Status:       Pending
Containers:
  api:
    Image:          myapp:v2.0.1          ← typo here?
    State:          Waiting
      Reason:       ImagePullBackOff
    Last State:     Terminated
      Reason:       OOMKilled             ← killed for using too much memory
      Exit Code:    137
    Requests:       cpu: 4, memory: 8Gi   ← too big for any node?
Events:                                    ← ⭐ READ THIS FIRST, from the bottom up
  Type     Reason     Message
  Warning  Failed     Failed to pull image "myapp:v2.0.1": not found
  Warning  BackOff    Back-off pulling image "myapp:v2.0.1"
```

> ⭐ **Events at the bottom are the single most useful debugging output in Kubernetes.** Scheduling failures, image pull errors, probe failures, volume mount errors: they all appear there.

---

## 4. `kubectl logs`: What Did the App Say?

```bash
kubectl logs <pod>                       # the container's stdout/stderr
kubectl logs -f <pod>                    # follow live
kubectl logs <pod> --previous            # ⭐ the run that CRASHED (essential for CrashLoopBackOff)
kubectl logs <pod> -c <container>        # pick a container in a multi-container Pod
kubectl logs <pod> --tail=50             # ➕ last 50 lines only
kubectl logs <pod> --since=10m           # ➕ only the last 10 minutes
kubectl logs deploy/web                  # resource-type/name: one Pod of the Deployment
kubectl logs -l app=web --all-containers # ➕ every Pod with that label
```

> **Note from class:** `kubectl logs <type>/<name>` (e.g. `deploy/web`, `rs/web-7d9f`) picks **one** Pod of that resource for you. Use `-l app=web` to see **all** of them.
>
> ⚠️ `logs` shows nothing if the container **never started** (ImagePullBackOff, Pending). There's no app output yet, so use `describe` instead.

---

## 5. `kubectl exec`: Look Inside

```bash
kubectl exec -it <pod> -- sh                    # interactive shell
kubectl exec -it <pod> -- bash                  # only if the image HAS bash
kubectl exec <pod> -- env                       # one-off command: check env vars from ConfigMaps/Secrets
kubectl exec <pod> -- ls /etc/config            # are the mounted files there?
kubectl exec <pod> -- cat /etc/resolv.conf      # DNS settings
kubectl exec -it <pod> -c sidecar -- sh         # a specific container
```
The `--` separates kubectl's own flags from the command run inside the container.

> ⚠️ **`bash: not found` / `executable file not found`:** small images (alpine, busybox) only have `sh`. **Distroless** images have **no shell at all**. ➕ For those, use `kubectl debug -it <pod> --image=busybox --target=<container>`, which attaches a temporary debug container.

**Test networking from inside the cluster:**
```bash
kubectl run tmp --rm -it --image=busybox:1.36 --restart=Never -- sh
/ # wget -qO- http://backend-service:80        # does the Service answer?
/ # nslookup backend-service                    # does DNS resolve it?
```

---

## 6. Events: The Cluster's Diary

```bash
kubectl get events                                   # events in the current namespace
kubectl get events --watch                           # stream new events live
kubectl get events --sort-by=.lastTimestamp          # ➕ oldest → newest (the default order is messy)
kubectl get events --field-selector type=Warning     # ➕ only problems
kubectl get events -A                                # ➕ all namespaces
kubectl events --for pod/<pod>                       # ➕ newer kubectl: events for one object
```

| `describe` Events | `get events` |
|-------------------|--------------|
| For **one** object | For **everything** in the namespace |
| "Why is **this Pod** broken?" | "What has been **going on** in the cluster?" (node problems, evictions, scaling) |

> ⚠️ Events are **kept only ~1 hour** by default. Investigate soon, or ship them to a logging system.

---

## 7. Common Errors & How to Fix Them

### 7.1 `CrashLoopBackOff`: the app starts, then dies, again and again
K8s restarts it with growing delays (10 s, 20 s, 40 s … up to 5 min).

| Common cause | How you spot it | Fix |
|--------------|-----------------|-----|
| App error / bad command / missing file | `logs --previous` shows the error | Fix the code, `command` or `args` |
| Missing env var or config | `logs` shows "DB_HOST not set" | Add it via a ConfigMap/Secret ([L11](11-Kubernetes-ConfigMaps-Secrets-Ingress.md#4-using-a-configmap-in-a-pod)) |
| Container's job finishes and it exits | Exit code `0`, logs look fine | `restartPolicy: Never`, or use a Job ([L8 §12](08-Kubernetes-Basics.md#12-hands-on-a-run-once-task-pod)) |
| **OOMKilled** ➕ | `describe` → `Last State: OOMKilled`, exit `137` | Raise `resources.limits.memory`, or fix the memory leak |
| Liveness probe kills it ➕ | Events: `Liveness probe failed` | Fix the probe path/port, or increase `initialDelaySeconds` |

```bash
kubectl logs <pod> --previous        # ⭐ step 1, always
kubectl describe pod <pod>           # exit code + Last State + Events
```

### 7.2 `ImagePullBackOff` / `ErrImagePull`: can't get the image
| Cause | Fix |
|-------|-----|
| Typo in the image name or tag (`ngnix`, `v2.0.1` doesn't exist) | Check that the tag exists in the registry, then correct it |
| **Private registry**, no credentials ➕ | Create a `docker-registry` Secret and add `imagePullSecrets` ([L11 §5](11-Kubernetes-ConfigMaps-Secrets-Ingress.md#5-secrets)) |
| Image built locally, not visible to the cluster ➕ | `minikube image load myapp:1.0`, or push it to a registry |
| Docker Hub rate limit / no internet on nodes ➕ | Events say `toomanyrequests` / timeout |

```bash
kubectl describe pod <pod>   # Events: "Failed to pull image ...: not found / unauthorized"
kubectl set image deploy/api api=myapp:2.0.0     # fix the tag
```

### 7.3 `Pending`: the Pod isn't placed on any node
The **scheduler** can't find a suitable node ([L8 §4](08-Kubernetes-Basics.md#4-control-plane-master-node-components)). `describe` Events say exactly why:

| Events message | Cause | Fix |
|----------------|-------|-----|
| `0/3 nodes are available: 3 Insufficient cpu/memory` | **Requests** are bigger than any node's free space | Lower `resources.requests`, or add nodes |
| `node(s) didn't match Pod's node affinity/selector` | `nodeSelector` / affinity points at labels no node has | Fix the selector, or label a node: `kubectl label node n1 disk=ssd` |
| `pod has unbound immediate PersistentVolumeClaims` ➕ | PVC is `Pending` | Fix the PVC ([L12 §8](12-Kubernetes-Volumes-and-Autoscaling.md#8-access-modes-reclaim-policies--pv-phases)) |
| `node(s) had untolerated taint` ➕ | Nodes are **tainted** (e.g. control-plane only) | Add a toleration, or use other nodes |

> ⚠️ **Correction to the class notes:** the scheduler looks at **requests**, not limits. "Adjust resource **limits**" won't fix a Pending Pod. Lower the **requests**.

### 7.4 ➕ More errors you'll meet
| STATUS / symptom | Meaning | Fix |
|------------------|---------|-----|
| `CreateContainerConfigError` | A referenced ConfigMap/Secret (or key) **doesn't exist** | Create it, or fix its name |
| `ContainerCreating` for minutes | Volume can't mount, or a slow image pull | `describe` → Events (`FailedMount`) |
| `Running` but `READY 0/1` | **Readiness probe** failing | Fix the app health endpoint or the probe |
| `Evicted` | The node ran out of memory/disk | Set requests/limits, clean up the node |
| Running fine, but the **Service doesn't answer** | Selector ≠ Pod labels, or wrong `targetPort` | `kubectl get endpoints <svc>` ([L9 §10](09-Kubernetes-Objects-Services-Deployments.md#10-port-mapping-containerport-targetport-port-nodeport)) |
| Ingress returns 404 / no ADDRESS | No controller, wrong host/path, or path not rewritten | [L11 §9–11](11-Kubernetes-ConfigMaps-Secrets-Ingress.md#9-ingress-vs-ingress-controller) |

---

## 8. Troubleshooting Flowchart

```
                    kubectl get pods
                           │
     ┌──────────────┬──────┴─────────┬──────────────────────┬────────────────────┐
     ▼              ▼                ▼                      ▼                    ▼
  Pending     ImagePullBackOff  CrashLoopBackOff   CreateContainerConfigError  Running but app
     │              │                │                      │                  not reachable
 describe pod   describe pod    logs --previous        describe pod                │
 (Events)       (Events)        + describe              (Events)           get endpoints <svc>
     │              │           (exit code)                 │                      │
 requests too   wrong name/tag, OOMKilled? → limits   missing ConfigMap/   empty → fix selector/labels
 big? selector? private repo?   app error? → fix code  Secret → create it   has IPs → check targetPort,
 PVC pending?   → fix image     job exits? → Job                            exec + wget from a test Pod
```

---

## 9. Hands-on: Break It & Fix It

```bash
# 1. CrashLoopBackOff: a command that fails
kubectl run crash --image=busybox -- sh -c "echo starting; ls /nope; exit 1"
kubectl get pods -w                        # Error → CrashLoopBackOff, RESTARTS climbing
kubectl logs crash --previous              # ls: /nope: No such file or directory  ← the cause
kubectl delete pod crash

# 2. ImagePullBackOff: a tag that doesn't exist
kubectl run badimg --image=nginx:9.99
kubectl describe pod badimg | tail -5      # Failed to pull image "nginx:9.99": ... not found
kubectl delete pod badimg

# 3. Pending: asking for more CPU than exists
kubectl run greedy --image=nginx --overrides='{"spec":{"containers":[{"name":"greedy","image":"nginx","resources":{"requests":{"cpu":"64"}}}]}}'
kubectl describe pod greedy | grep -A3 Events   # 0/1 nodes are available: 1 Insufficient cpu
kubectl delete pod greedy

# 4. Watch all of it from the cluster's point of view
kubectl get events --sort-by=.lastTimestamp | tail -20
```

---

## 10. Helm: Why a Package Manager?

A real app is **many YAML files**: Deployment, Service, Ingress, ConfigMap, Secret, HPA, PVC … and **each environment needs slightly different values** (dev: 1 replica, `latest` tag; prod: 5 replicas, a pinned tag, a real domain).

**Without Helm:** copy-paste YAML per environment, edit by hand, `kubectl apply` 8 files in the right order, and remember what you deployed so you can remove it later.

**With Helm:** the YAMLs become **templates** with placeholders. You fill them from a small **values** file, then install, upgrade, roll back or uninstall the **whole app with one command**.

| Linux | Kubernetes |
|-------|------------|
| `apt` / `yum` / `brew` | **Helm** |
| `.deb` / `.rpm` package | **Chart** |
| Package repository | **Chart repository** (Artifact Hub, Bitnami, …) |
| `apt install nginx` | `helm install my-nginx bitnami/nginx` |

> 💡 **Simple words mein:** Helm = Kubernetes ka Play Store. Chart = app ka package. Ek command se poora app (saari YAML files) install, update ya uninstall.

**What Helm gives you:**
- **Templating:** one chart, many environments (only the values change).
- **Releases:** Helm remembers what it installed, so `helm uninstall` removes **everything** it created.
- **Versioned upgrades and one-command rollback.**
- **Reuse:** install ready-made charts (PostgreSQL, Redis, Prometheus, ingress-nginx) instead of writing them.

> ⚠️ **Correction:** the class notes say Helm automates "scaling". It doesn't scale anything at run time. That's the **HPA** ([L12 §10](12-Kubernetes-Volumes-and-Autoscaling.md#10-horizontal-pod-autoscaler-hpa)). Helm only **packages and deploys**: you can set `replicaCount` or enable an HPA template through values.

---

## 11. Helm Key Terms

| Term | Meaning |
|------|---------|
| **Chart** | A **package**: a folder of templates + default values + metadata |
| **Values** | The **configuration** that fills the templates (`values.yaml`, `-f file`, `--set`) |
| **Release** | One **installed instance** of a chart in a cluster, with a name you choose. The same chart can be installed many times (`shop-dev`, `shop-prod`). |
| **Revision** | A version of a release. Each install/upgrade/rollback adds one (1, 2, 3 …) |
| **Repository** | A place that hosts charts (an HTTP index or an OCI registry) |

```
 Chart (templates + values.yaml)  +  your values  ──helm install──▶  Release "my-nginx" (revision 1)
                                                                      = rendered YAML applied to the cluster
```

> ➕ **Helm 3** (current) is just a **client CLI**. It talks to the API server using your kubeconfig, and stores release history as **Secrets** in the namespace. The old server-side component **Tiller** (Helm 2) is gone.

---

## 12. Helm Basic Commands

### Find & add charts
```bash
helm repo add bitnami https://charts.bitnami.com/bitnami   # ⚠️ needed before "bitnami/nginx" works
helm repo update                                           # refresh the chart index (like apt update)
helm search repo nginx                                     # search added repos
helm search hub nginx                                      # ➕ search Artifact Hub (public)
helm show values bitnami/nginx                             # ➕ every setting you can change
```
> ➕ Many charts are also published to **OCI registries** and can be installed straight from them, with no `repo add`: `helm install my-nginx oci://registry-1.docker.io/bitnamicharts/nginx`.

### Install, list, uninstall (from class)
```bash
helm install my-nginx bitnami/nginx                        # helm install <release-name> <chart>
helm list                                                  # releases in this namespace (-A for all)
helm uninstall my-nginx                                    # removes EVERY object the release created
```

### ➕ Inspect a release
```bash
helm status my-nginx                # state + the chart's NOTES (how to access the app)
helm get values my-nginx            # values YOU supplied
helm get manifest my-nginx          # the exact YAML that was applied
helm history my-nginx               # list of revisions
kubectl get all -l app.kubernetes.io/instance=my-nginx   # its objects via the standard label
```

### ➕ Customising at install time
```bash
helm install my-nginx bitnami/nginx --set replicaCount=3 --set service.type=NodePort
helm install my-nginx bitnami/nginx -f my-values.yaml
helm install my-nginx bitnami/nginx -n web --create-namespace     # into a namespace
```
**Priority (lowest → highest):** chart's `values.yaml` → `-f file.yaml` (later files win) → `--set`.

---

## 13. Creating a Custom Chart

```bash
helm create demo-chart
```
generates a working chart (it deploys nginx by default):
```
demo-chart/
├── Chart.yaml            # metadata: name, chart version, app version
├── values.yaml           # default configuration values
├── charts/               # ➕ dependencies (sub-charts), e.g. a postgres chart
├── .helmignore           # ➕ files to leave out when packaging (like .gitignore)
└── templates/            # Kubernetes YAML with {{ placeholders }}
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml          # off by default (ingress.enabled: false)
    ├── hpa.yaml              # off by default (autoscaling.enabled: false)
    ├── serviceaccount.yaml
    ├── _helpers.tpl          # ➕ reusable snippets (names, labels). "_" = not rendered as a file
    ├── NOTES.txt             # ➕ message printed after install
    └── tests/
        └── test-connection.yaml   # ➕ run with: helm test <release>
```

### `Chart.yaml`
```yaml
apiVersion: v2              # v2 = Helm 3 chart
name: demo-chart
description: A Helm chart for Kubernetes
type: application
version: 0.1.0              # the CHART's version: bump it when templates change
appVersion: "1.16.0"        # the APP's version (often used as the default image tag)
```

### `values.yaml` (excerpt)
```yaml
replicaCount: 1
image:
  repository: nginx
  tag: ""                   # empty → falls back to Chart.appVersion
  pullPolicy: IfNotPresent
service:
  type: ClusterIP
  port: 80
resources: {}
autoscaling:
  enabled: false
  minReplicas: 1
  maxReplicas: 100
  targetCPUUtilizationPercentage: 80
```

---

## 14. Templates & Values: How Helm Works

Templates are normal Kubernetes YAML with **Go template** placeholders `{{ }}`. At install time, Helm **renders** them with the values, then sends the result to the API server.

**`templates/deployment.yaml` (simplified):**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ .Release.Name }}-web                    # → demo-release-web
spec:
  replicas: {{ .Values.replicaCount }}             # → 1  (from values.yaml)
  selector:
    matchLabels:
      app: {{ .Chart.Name }}
  template:
    metadata:
      labels:
        app: {{ .Chart.Name }}
    spec:
      containers:
        - name: web
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          ports:
            - containerPort: 80
          {{- if .Values.resources }}               # ➕ conditional block
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          {{- end }}
```

| Built-in object | Gives you |
|-----------------|-----------|
| `.Values` | Everything from `values.yaml`, `-f` and `--set` |
| `.Release.Name` / `.Release.Namespace` | The release name / namespace you install into |
| `.Chart.Name` / `.Chart.Version` / `.Chart.AppVersion` | Fields from `Chart.yaml` |

> 💡 **Simple words mein:** template = form jisme khaali jagah hai (`{{ .Values.replicaCount }}`). values.yaml = form bharne ka data. Helm dono milaake final YAML bana deta hai.

### ➕ See the YAML without installing (great for debugging charts)
```bash
helm lint ./demo-chart                                  # check the chart for mistakes
helm template demo-release ./demo-chart                 # print the rendered YAML locally
helm install demo-release ./demo-chart --dry-run --debug   # render + validate against the cluster
```

---

## 15. Upgrade, Rollback & Environments

```bash
helm upgrade demo-release ./demo-chart --set replicaCount=3   # change config/version → revision 2
helm history demo-release
# REVISION  STATUS      DESCRIPTION
# 1         superseded  Install complete
# 2         deployed    Upgrade complete
helm rollback demo-release 1                                  # back to revision 1 → creates revision 3
helm upgrade --install demo-release ./demo-chart              # ➕ install if missing, else upgrade (CI/CD favourite)
```
The rollback idea is the same as `kubectl rollout undo` ([L10 §9](10-Kubernetes-Deployment-Strategies.md#9-worked-example-rolling-update--verify--roll-back)), but for **the whole app** (Deployment + Service + ConfigMap + …), not just one Deployment.

### One chart, many environments
```
demo-chart/
├── values.yaml           # defaults
├── values-dev.yaml       # replicaCount: 1, image.tag: latest
└── values-prod.yaml      # replicaCount: 5, image.tag: "2.3.1", ingress.enabled: true
```
```bash
helm upgrade --install shop-dev  ./demo-chart -f values-dev.yaml  -n dev
helm upgrade --install shop-prod ./demo-chart -f values-prod.yaml -n prod
```

---

## 16. Hands-on: `demo-chart` End to End

```bash
# 1. Create the chart
helm create demo-chart

# 2. Customise values.yaml: e.g. replicaCount: 2, service.type: NodePort
#    (or leave it as-is: it deploys nginx out of the box)

# 3. Check before installing
helm lint ./demo-chart
helm template demo-release ./demo-chart | less

# 4. Install
helm install demo-release ./demo-chart

# 5. Verify
helm list                                       # demo-release   deployed   demo-chart-0.1.0
kubectl get all -l app.kubernetes.io/instance=demo-release
helm status demo-release                        # prints NOTES.txt with access instructions

# 6. Upgrade, then roll back
helm upgrade demo-release ./demo-chart --set replicaCount=3
kubectl get pods                                # 3 Pods
helm rollback demo-release 1
kubectl get pods                                # back to the count in values.yaml (1 by default)

# 7. Clean up: one command removes everything
helm uninstall demo-release
```

---

## 17. Quick Revision Cheat Sheet

```
ORDER         get (what) → describe (why, EVENTS) → logs (app said) → exec (inside) → get events (cluster)
GET           get pods [-w watch] [-o wide node/IP] [-A] [--show-labels] · get pod x -o yaml
DESCRIBE      Events at bottom ⭐ · State / Last State (OOMKilled, exit code) · Requests
LOGS          logs pod · -f · --previous ⭐ · -c ctr · --tail · --since · logs deploy/x · -l app=x
EXEC          exec -it pod -- sh (bash only if present) · exec pod -- env · kubectl debug for distroless ➕
EVENTS        get events [--watch] [--sort-by=.lastTimestamp] [--field-selector type=Warning] · ~1 h retention
CRASHLOOP     app keeps dying → logs --previous · bad cmd / missing config / OOMKilled 137 / liveness / job exits
IMAGEPULL     bad name/tag · private repo → imagePullSecrets · local image → minikube image load
PENDING       Insufficient cpu/mem (lower REQUESTS, not limits) · nodeSelector/affinity · PVC pending · taints
OTHER ➕      CreateContainerConfigError = missing ConfigMap/Secret · READY 0/1 = readiness · svc → get endpoints
HELM          package manager for K8s (apt for K8s) · chart = package · release = installed instance · revision
REPO          helm repo add bitnami <url> · repo update · search repo|hub · show values
CORE          helm install <release> <chart> · helm list · helm uninstall <release>
CUSTOMISE     --set k=v · -f values.yaml · priority: chart values < -f < --set
CHART         helm create x → Chart.yaml (version, appVersion) · values.yaml · templates/ · charts/ · _helpers.tpl · NOTES.txt
TEMPLATE      {{ .Values.x }} {{ .Release.Name }} {{ .Chart.Name }} · helm lint · helm template · --dry-run --debug
LIFECYCLE     helm upgrade · helm history · helm rollback <rel> <rev> · helm upgrade --install
ENVS          values-dev.yaml / values-prod.yaml with -f · Helm ≠ autoscaling (that's HPA)
```

---

## 18. Likely Exam / Interview Questions

1. **What's your step-by-step approach to debugging a failing Pod?** `get` (status) → `describe` (Events) → `logs` (`--previous` if it's crashing) → `exec` (inspect inside) → `get events` (cluster-wide).
2. **What does `kubectl get pods -w` do?** It watches, printing an updated line whenever a Pod's status changes.
3. **What's the most useful part of `kubectl describe pod`?** The Events section at the bottom, which shows scheduling, image pull, mount and probe failures.
4. **How do you see logs from a container that crashed and was restarted?** `kubectl logs <pod> --previous`.
5. **Why might `kubectl logs` show nothing?** The container never started (ImagePullBackOff, Pending), so check `describe`. It could also be the wrong container in a multi-container Pod (use `-c`).
6. **`kubectl exec -it pod -- bash` fails with "executable file not found". Why?** The image has no bash. Use `sh`, or `kubectl debug` for distroless images.
7. **`describe` Events vs. `kubectl get events`?** Events for one object vs. every event in the namespace or cluster. Events are kept for about an hour.
8. **What causes CrashLoopBackOff, and how do you debug it?** The container keeps exiting: an app error, a bad command, missing config, OOMKilled, or a failing liveness probe. Check `logs --previous` and the exit code in `describe`.
9. **A container shows `Last State: OOMKilled, exit code 137`. What happened, and what's the fix?** It went over its memory limit and was killed. Raise `limits.memory` or fix the leak.
10. **What causes ImagePullBackOff?** A wrong image name or tag, a private registry without `imagePullSecrets`, or an image that only exists on your laptop.
11. **A Pod is stuck in Pending. What are the likely causes?** Requests too big for any node, a nodeSelector or affinity that matches no node, an unbound PVC, or taints.
12. **Does lowering resource limits fix a Pod that is Pending for "Insufficient cpu"?** No. The scheduler uses requests, so lower the requests or add capacity.
13. **What is Helm, and why use it?** A package manager for Kubernetes. It templates many manifests into one chart, configures it per environment through values, and installs, upgrades, rolls back and uninstalls the whole app as a release.
14. **Chart vs. release vs. revision?** A chart is the package. A release is an installed instance of it. A revision is a version of that release.
15. **What files does `helm create` generate?** `Chart.yaml` (metadata), `values.yaml` (defaults), `templates/` (YAML with placeholders), `charts/` (dependencies), plus `_helpers.tpl`, `NOTES.txt` and tests.
16. **`version` vs. `appVersion` in Chart.yaml?** `version` is the chart package's version. `appVersion` is the version of the application it deploys.
17. **How do you override values at install time, and which source wins?** `-f values.yaml` and `--set key=val`. `--set` beats `-f`, which beats the chart's defaults.
18. **How do you see the YAML a chart will produce without installing it?** `helm template <release> <chart>`, or `helm install --dry-run --debug`.
19. **How do you roll back a Helm release?** `helm history <release>`, then `helm rollback <release> <revision>`.
20. **`helm install my-nginx bitnami/nginx` fails with "repo bitnami not found". Why?** The repository wasn't added. Run `helm repo add bitnami <url>` and `helm repo update`.
