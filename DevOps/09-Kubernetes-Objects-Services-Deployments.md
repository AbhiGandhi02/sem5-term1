# Kubernetes Part 2: Labels, Services, ReplicaSets, Deployments & DaemonSets

> **DevOps – Lecture 9 notes.** Anything marked **➕ Extra** was not covered in class. It's there because it fills a gap or comes up often in labs and interviews.
>
> **Builds on:** [Lecture 8 – Kubernetes Basics](08-Kubernetes-Basics.md) (architecture, YAML fields, Pods, the ReplicaSet controller). Port ideas come from [Lecture 3](03-Networking.md#3-ports) and [Lecture 5](05-Docker.md#9-port-mapping--docker-networking).
>
> **Practice:** [Kubernetes MCQs](Kubernetes-MCQs.md) · **Next:** [Lecture 10 – Deployment Strategies](10-Kubernetes-Deployment-Strategies.md).

---

## Table of Contents
1. [The Big Picture: How the Objects Fit Together](#1-the-big-picture-how-the-objects-fit-together)
2. [Everyday kubectl Commands](#2-everyday-kubectl-commands)
3. [Namespaces](#3-namespaces)
4. [Labels & Selectors](#4-labels--selectors)
5. [ReplicaSets](#5-replicasets)
6. [Deployments](#6-deployments)
7. [Rolling Updates: `maxSurge` & `maxUnavailable`](#7-rolling-updates-maxsurge--maxunavailable)
8. [Services: Why We Need Them](#8-services-why-we-need-them)
9. [Service Types](#9-service-types)
10. [Port Mapping: `containerPort`, `targetPort`, `port`, `nodePort`](#10-port-mapping-containerport-targetport-port-nodeport)
11. [DaemonSets](#11-daemonsets)
12. [Hands-on Lab: Deploy, Expose, Scale, Update, Roll Back](#12-hands-on-lab-deploy-expose-scale-update-roll-back)
13. [Quick Revision Cheat Sheet](#13-quick-revision-cheat-sheet)
14. [Likely Exam / Interview Questions](#14-likely-exam--interview-questions)

---

## 1. The Big Picture: How the Objects Fit Together

```
                     ┌──────────────── Service (stable IP + DNS name) ────────────────┐
                     │      selector: app=web   → load-balances to matching Pods      │
                     └───────────────┬──────────────────┬──────────────────┬──────────┘
                                     ▼                  ▼                  ▼
 Deployment ──owns──▶ ReplicaSet ──owns──▶ [Pod app=web] [Pod app=web] [Pod app=web]
 (versions, rolling   (keeps exactly N)
  updates, rollback)

 DaemonSet ──▶ exactly ONE Pod on EVERY node (logging / monitoring agents)
```

| Object | One-line job |
|--------|--------------|
| **Pod** | Runs your container(s). Smallest unit. Disposable. |
| **ReplicaSet** | Keeps **N identical Pods** running |
| **Deployment** | Manages ReplicaSets, giving you **updates without downtime + rollback** |
| **Service** | A **stable address** in front of changing Pods |
| **DaemonSet** | **One Pod per node** |
| **Labels / selectors** | The **glue** that connects all of the above |

> 💡 **Simple words mein:** Pods aate-jaate rehte hain (IP badalta rehta hai). Deployment unhe zinda rakhta hai, Service unke aage ek fixed "reception" number deta hai, aur labels se sab ek-dusre ko pehchaante hain.

---

## 2. Everyday kubectl Commands

| Command | Shows |
|---------|-------|
| `kubectl get pods` | Pods in the **current namespace** (all phases, not just running) |
| `kubectl get pods -A` ➕ | Pods in **all** namespaces |
| `kubectl get namespaces` (`ns`) | Namespaces in the cluster |
| `kubectl get nodes` | Nodes and whether they're `Ready` |
| `kubectl get replicasets` (`rs`) | ReplicaSets: DESIRED / CURRENT / READY counts |
| `kubectl get deployments` (`deploy`) | Deployments: READY / UP-TO-DATE / AVAILABLE |
| `kubectl get services` (`svc`) | Services: type, cluster IP, ports |
| `kubectl get daemonsets` (`ds`) | DaemonSets |
| `kubectl get all` ➕ | Pods, Services, Deployments, ReplicaSets together |
| `kubectl describe <type> <name>` | Full details + **Events** of any object, e.g. `kubectl describe node minikube` |

> Short names (`po`, `rs`, `deploy`, `svc`, `ns`, `ds`) all work. ➕ `kubectl api-resources` lists every type with its short name.

---

## 3. Namespaces

A **namespace** is a **virtual folder inside the cluster**. It separates teams or environments (dev / staging / prod) on the same cluster. Names must be unique **within** a namespace, not across the whole cluster.

| Built-in namespace | Holds |
|--------------------|-------|
| `default` | Your objects, if you don't choose a namespace |
| `kube-system` | Kubernetes' own components (DNS, kube-proxy, …) |
| `kube-public` / `kube-node-lease` ➕ | Public cluster info / node heartbeats |

```bash
kubectl create namespace dev
kubectl apply -f app.yaml -n dev        # create objects in "dev"
kubectl get pods -n kube-system         # look inside a namespace
kubectl config set-context --current --namespace=dev   # ➕ make "dev" your default
```

---

## 4. Labels & Selectors

### Labels
**Key/value tags** attached to objects (mostly Pods). They don't do anything by themselves. They're for **identifying and grouping**.
```yaml
metadata:
  name: web-7d9f
  labels:
    app: web
    env: prod
    tier: frontend
```

### Selectors
A **selector** is a **query on labels**: "give me every Pod with `app=web`". ReplicaSets, Deployments and Services all use selectors to find **their** Pods.

> 💡 **Simple words mein:** label = kapdon pe laga tag ("size: M, color: blue"). Selector = filter ("mujhe saare blue M dikhao"). Hazaron Pods mein se sahi wale isi se milte hain.

```bash
kubectl get pods --show-labels                 # see labels
kubectl get pods -l app=web                    # equality selector
kubectl get pods -l 'env in (prod, staging)'   # ➕ set-based selector
kubectl get pods -l app=web,env=prod           # AND
kubectl label pod web-7d9f version=v2          # ➕ add a label to a live Pod
```

| In YAML | Selector style |
|---------|----------------|
| Service | `selector: { app: web }` (simple key/value) |
| ReplicaSet / Deployment | `selector: { matchLabels: { app: web } }` (➕ also `matchExpressions`) |

> ⚠️ **The #1 Kubernetes bug:** a selector that doesn't match the Pod labels (a typo like `app: Web` vs. `app: web`).
> - In a Deployment, the API server **rejects** it ([MCQ Q8](Kubernetes-MCQs.md)).
> - In a Service, it's **accepted silently**, but the Service sends traffic nowhere. Check with `kubectl get endpoints <svc>`: empty = no matching Pods.

---

## 5. ReplicaSets

A **ReplicaSet** keeps a **specified number of identical Pods** running at all times. A Pod is deleted, crashes, or its node dies → it creates a replacement. One too many → it deletes one.

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: web-rs
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web            # "my Pods are the ones labelled app=web"
  template:               # blueprint for new Pods
    metadata:
      labels:
        app: web          # MUST match the selector
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
```

```bash
kubectl apply -f rs.yaml
kubectl get rs                                 # NAME     DESIRED   CURRENT   READY
                                               # web-rs   3         3         3
kubectl scale rs web-rs --replicas=5           # scale up (or edit replicas: in YAML and re-apply)
kubectl delete pod <one-web-rs-pod>            # watch a replacement appear: kubectl get pods -w
```

> ⚠️ **Gotcha:** if the ReplicaSet was **created by a Deployment**, scaling the ReplicaSet does nothing lasting. The Deployment puts the old count straight back. **Scale the Deployment instead:** `kubectl scale deploy web --replicas=5`.
>
> In practice you **rarely write ReplicaSets yourself**. You write a Deployment, and it creates them.

---

## 6. Deployments

A **Deployment** gives **declarative updates** for Pods and ReplicaSets. You change the YAML (e.g. a new image tag), and the Deployment moves the cluster from the old version to the new one **safely**.

**What it adds on top of a ReplicaSet:**
| Feature | How |
|---------|-----|
| **Rolling updates** (zero downtime) | Creates a **new ReplicaSet** for the new version, scales it **up** while scaling the old one **down** |
| **Rollback** | Old ReplicaSets are kept (scaled to 0), so going back = scaling the old one up again |
| **Revision history** | `kubectl rollout history` |
| **Pause / resume** ➕ | Batch several changes into one rollout |

```
 Update image nginx:1.14 → nginx:1.27

 Deployment
   ├── ReplicaSet web-5d8f (nginx:1.14)  3 → 2 → 1 → 0   (kept for rollback)
   └── ReplicaSet web-7c4b (nginx:1.27)  0 → 1 → 2 → 3
```

**Corrected example from class** (the class version was missing `selector` and the template `labels`, so the API server would reject it):
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example-deployment
spec:
  replicas: 3
  selector:                 # required for apps/v1
    matchLabels:
      app: example
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1           # at most 3 + 1 = 4 Pods at any moment
      maxUnavailable: 0     # never fewer than 3 ready Pods
  template:
    metadata:
      labels:
        app: example        # must match the selector
    spec:
      containers:
        - name: nginx
          image: nginx:1.14.2
          ports:
            - containerPort: 80
```

---

## 7. Rolling Updates: `maxSurge` & `maxUnavailable`

| Setting | Meaning | Default |
|---------|---------|---------|
| **`maxSurge`** | How many **extra** Pods **above** `replicas` may exist during the update | 25% |
| **`maxUnavailable`** | How many Pods **below** `replicas` may be unavailable during the update | 25% |

Both take a **number** (`1`) or a **percentage** (`25%`). They **can't both be 0**, because then the update could never make progress.

### Walk-through: `replicas: 3`, `maxSurge: 1`, `maxUnavailable: 0`
```
 Allowed range: min 3 ready, max 4 total

 Step   Old (v1)   New (v2)   Total   What happened
  0       3          0          3     start
  1       3          1          4     surge: create 1 new Pod (max 4)
  2       2          1          3     new Pod is Ready → delete 1 old
  3       2          2          4     create another new
  4       1          2          3     delete another old
  5       1          3          4     ...
  6       0          3          3     done ✅  never below 3 ready = zero downtime
```

| Combination | Behaviour | When |
|-------------|-----------|------|
| `maxSurge: 1, maxUnavailable: 0` | Slow, **never loses capacity**, needs spare room for 1 extra Pod | Production default choice for safety |
| `maxSurge: 0, maxUnavailable: 1` | **No extra Pods**, temporarily runs at N−1 | Cluster is short on CPU/memory |
| `maxSurge: 100%, maxUnavailable: 0` | Brings up a full new set at once, then drops the old | Fastest safe swap, but needs double resources |

> 💡 **Simple words mein:** maxSurge = "kitne extra log temporarily bula sakte ho". maxUnavailable = "kitne log ek saath chhutti pe ja sakte hain". Dono 0 = koi aa bhi nahi sakta, ja bhi nahi sakta, update atak jaayega.

**➕ The other strategy, `Recreate`:** kill **all** old Pods, then start the new ones. There's **downtime**, but the two versions never run together (useful when they can't share a database schema).

### Rollout commands
```bash
kubectl set image deploy/example-deployment nginx=nginx:1.27   # trigger an update (or edit YAML + apply)
kubectl rollout status deploy/example-deployment               # watch it progress
kubectl rollout history deploy/example-deployment              # list revisions
kubectl rollout undo deploy/example-deployment                 # roll back to the previous revision
kubectl rollout undo deploy/example-deployment --to-revision=1 # ➕ to a specific one
kubectl get rs                                                 # see old (0 replicas) + new ReplicaSets
```

> ➕ A rolling update is only truly zero-downtime if K8s knows **when a new Pod is ready**. Add a **readinessProbe** (e.g. an HTTP check on `/health`) so traffic only goes to Pods that can actually serve it.

---

## 8. Services: Why We Need Them

**Problem:** Pods are **disposable**. Each new Pod gets a **new IP**. If a frontend hard-codes a backend Pod's IP, it breaks the moment that Pod is replaced or scaled.

**Solution: a Service**
- A **stable virtual IP** (ClusterIP) and a **stable DNS name** that never change.
- Uses a **selector** to find the current matching Pods (its **endpoints**) and **load-balances** traffic across them.
- **kube-proxy** on each node does the actual routing ([L8 §5](08-Kubernetes-Basics.md#5-worker-node-components)).

```
 frontend Pod ──▶ http://backend:80 ──▶ Service "backend" (10.96.12.7, never changes)
                                             ├──▶ Pod 10.244.1.5:8080
                                             ├──▶ Pod 10.244.2.9:8080
                                             └──▶ Pod 10.244.1.8:8080  (Pods come and go)
```

> **➕ Service DNS:** inside the cluster, reach a Service at `backend` (same namespace) or `backend.<namespace>.svc.cluster.local` (any namespace). It's the same idea as Docker's container-name DNS ([L7 §3.1](07-Docker-Networking-and-Volumes.md#31-bridge-default)).

---

## 9. Service Types

| Type | Reachable from | How | Use |
|------|----------------|-----|-----|
| **ClusterIP** (default) | **Inside the cluster only** | Internal virtual IP | Service-to-service traffic (backend, DB) |
| **NodePort** | Outside, via `<any-node-IP>:<nodePort>` | Opens the **same port (30000–32767)** on **every node** | Testing, bare-metal clusters |
| **LoadBalancer** | Internet, via a cloud LB's public IP | Asks AWS/GCP/Azure for a **real load balancer** that forwards to the NodePorts | Exposing an app publicly on the cloud |
| **ExternalName** | Inside the cluster | A **DNS alias (CNAME)** to an outside name. No selector, no proxying. | Point `db` at `mydb.xyz.rds.amazonaws.com` |
| **Headless** (`clusterIP: None`) | Inside the cluster | **No virtual IP.** DNS returns the **individual Pod IPs**. | Direct Pod access: databases, StatefulSets |

**They build on each other:**
```
 LoadBalancer  =  NodePort + cloud load balancer
 NodePort      =  ClusterIP + a port opened on every node
 ClusterIP     =  internal virtual IP
```

> 💡 **Simple words mein:** ClusterIP = office ka internal extension number (sirf andar se lagta hai). NodePort = har branch ka ek fixed public number. LoadBalancer = ek main helpline jo calls sab branches mein baant deti hai.

### YAML examples
```yaml
# ClusterIP (default: no "type" line needed)
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  selector:
    app: backend
  ports:
    - port: 80            # Service port
      targetPort: 8080    # container port on the Pods
---
# NodePort
apiVersion: v1
kind: Service
metadata:
  name: web-nodeport
spec:
  type: NodePort
  selector:
    app: web
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080     # optional: omit it and K8s picks one in 30000–32767
---
# ExternalName
apiVersion: v1
kind: Service
metadata:
  name: db
spec:
  type: ExternalName
  externalName: mydb.abc123.ap-south-1.rds.amazonaws.com
---
# Headless
apiVersion: v1
kind: Service
metadata:
  name: mongo
spec:
  clusterIP: None
  selector:
    app: mongo
  ports:
    - port: 27017
```

> ➕ Quick way without YAML: `kubectl expose deploy web --port=80 --target-port=80 --type=NodePort`
> ➕ **minikube:** `LoadBalancer` stays `<pending>` (there's no cloud). Use `minikube service web --url` or `minikube tunnel`.

---

## 10. Port Mapping: `containerPort`, `targetPort`, `port`, `nodePort`

```
 Outside world                Node                     Service                  Pod / container
 ─────────────            ─────────────            ─────────────            ─────────────────
 192.168.49.2:30080  ──▶   nodePort 30080   ──▶   port 80 (ClusterIP) ──▶  targetPort 8080
                                                                            = containerPort 8080
                                                                            (where the app listens)
```

| Field | Lives in | Meaning |
|-------|----------|---------|
| **`containerPort`** | Pod spec | Port the app **inside the container** listens on (mostly documentation, like Docker's `EXPOSE`) |
| **`targetPort`** | Service | Port **on the Pod** the Service forwards to. **Must equal the port the app listens on.** If left out, it defaults to `port`. |
| **`port`** | Service | The Service's own port. Other Pods call `backend:<port>`. |
| **`nodePort`** | Service (NodePort / LB) | Port opened on **every node** for outside traffic (30000–32767) |

> 💡 **Simple words mein:** chain yaad rakho: **nodePort → port → targetPort = containerPort**. Bahar se node pe aaye, node se service pe, service se pod ke andar app pe.
>
> ⚠️ Most common mistake: `targetPort` ≠ the port the app really listens on → the connection is refused even though the Service and Pods look healthy.

### Port forwarding (quick local testing)
```bash
kubectl port-forward svc/my-service 8080:80     # localhost:8080 → Service port 80
kubectl port-forward pod/web-7d9f 8080:80       # ➕ straight to one Pod
kubectl port-forward deploy/web 8080:80         # ➕ to one Pod of a Deployment
```
It creates a **temporary tunnel from your laptop** through the API server. You don't need a NodePort or LoadBalancer, and it stops when you press Ctrl+C. **Use it for debugging, not production.**

---

## 11. DaemonSets

A **DaemonSet** makes sure **exactly one copy of a Pod runs on every node** (or every node matching a filter).
- A **new node joins** → it gets the Pod automatically.
- A node is **removed** → its Pod is cleaned up.
- There's **no `replicas` field**. The number of Pods = the number of nodes.

**Use for node-level agents:**
| Agent type | Examples |
|------------|----------|
| Log collection | Fluentd, Fluent Bit, Filebeat |
| Monitoring | Prometheus node-exporter, Datadog agent |
| Networking / storage | kube-proxy (often a DaemonSet itself ➕), CNI plugins like Calico |

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-logger
spec:
  selector:
    matchLabels:
      app: node-logger
  template:
    metadata:
      labels:
        app: node-logger
    spec:
      containers:
        - name: logger
          image: fluent/fluent-bit:3.0
```
```bash
kubectl get ds                       # DESIRED = CURRENT = number of nodes
kubectl get ds -n kube-system        # ➕ see the DaemonSets K8s itself runs
```

| | Deployment | DaemonSet |
|-|------------|-----------|
| How many Pods | `replicas: N`, placed wherever the scheduler likes | **One per node** |
| Grows when | You scale it | A **node is added** |
| Use for | Your apps | Per-node agents |

---

## 12. Hands-on Lab: Deploy, Expose, Scale, Update, Roll Back

```bash
# 1. Deploy (YAML from §6 saved as deploy.yaml)
kubectl apply -f deploy.yaml
kubectl get deploy,rs,pods --show-labels

# 2. Expose it with a Service
kubectl expose deploy example-deployment --port=80 --type=NodePort --name=example-svc
kubectl get svc example-svc                  # note the 3xxxx nodePort
kubectl get endpoints example-svc            # 3 Pod IPs = selector works ✅
kubectl port-forward svc/example-svc 8080:80 # open http://localhost:8080

# 3. Scale
kubectl scale deploy example-deployment --replicas=5
kubectl get endpoints example-svc            # now 5 Pod IPs, the Service picked them up automatically

# 4. Rolling update, and watch it
kubectl set image deploy/example-deployment nginx=nginx:1.27
kubectl rollout status deploy/example-deployment
kubectl get rs                               # old RS → 0, new RS → 5

# 5. Roll back
kubectl rollout history deploy/example-deployment
kubectl rollout undo deploy/example-deployment

# 6. Clean up
kubectl delete svc example-svc
kubectl delete -f deploy.yaml
```

---

## 13. Quick Revision Cheat Sheet

```
HIERARCHY     Deployment → ReplicaSet → Pods        Service → (selector) → Pods      DaemonSet → 1 Pod/node
KUBECTL       get pods|ns|nodes|rs|deploy|svc|ds|all   -A all namespaces   -n <ns>   describe <type> <name>
NAMESPACES    virtual folders · default · kube-system · names unique per namespace
LABELS        key: value tags in metadata.labels      SELECTOR = query: -l app=web
              mismatch → Deployment rejected · Service silently empty (check endpoints)
REPLICASET    keeps exactly N Pods · kubectl scale rs x --replicas=N (but scale the DEPLOY if it owns it)
DEPLOYMENT    new image → new RS scaled up, old RS scaled down (kept at 0 for rollback)
              apps/v1 needs selector.matchLabels == template.metadata.labels
ROLLING       maxSurge = extra Pods above replicas · maxUnavailable = Pods allowed below · default 25%/25%
              surge 1 / unavail 0 → never lose capacity · both 0 = not allowed · Recreate = downtime
ROLLOUT       set image · rollout status · rollout history · rollout undo [--to-revision=N]
SERVICE       stable IP + DNS name in front of changing Pods · kube-proxy routes · svc.ns.svc.cluster.local
TYPES         ClusterIP (default, internal) · NodePort (30000-32767 on every node)
              LoadBalancer (cloud LB → NodePort) · ExternalName (CNAME) · Headless (clusterIP: None → Pod IPs)
PORTS         nodePort → port (Service) → targetPort (Pod) = containerPort (app)
PORT-FWD      kubectl port-forward svc/x 8080:80 → temporary local tunnel for testing
DAEMONSET     one Pod per node, auto on new nodes, no replicas · logging / monitoring agents
```

---

## 14. Likely Exam / Interview Questions

1. **What are labels and selectors, and why do they matter?** Labels are key/value tags on objects. Selectors query them. They're how ReplicaSets, Deployments and Services find their Pods among thousands.
2. **ReplicaSet vs. Deployment?** A ReplicaSet only keeps N Pods alive. A Deployment manages ReplicaSets and adds rolling updates, revision history and rollback.
3. **You scale a Deployment's ReplicaSet to 5, but it goes back to 3. Why?** The Deployment owns the ReplicaSet and restores its own `replicas` value. Scale the Deployment instead.
4. **How does a rolling update work internally?** A new ReplicaSet is scaled up while the old one is scaled down, within the `maxSurge` and `maxUnavailable` limits. The old ReplicaSet is kept at 0 for rollback.
5. **With `replicas: 4`, `maxSurge: 1`, `maxUnavailable: 1`, what are the most total Pods and the fewest ready Pods during an update?** At most 5 Pods in total, and at least 3 ready.
6. **Why can't `maxSurge` and `maxUnavailable` both be 0?** The update could never create a new Pod or remove an old one, so it would never progress.
7. **RollingUpdate vs. Recreate?** RollingUpdate replaces Pods gradually with no downtime. Recreate kills all old Pods first, which causes downtime but never runs two versions together.
8. **How do you roll back a bad release?** `kubectl rollout undo deploy/<name>`, with `--to-revision=N` for a specific revision.
9. **Why do we need a Service if Pods already have IPs?** Pod IPs change whenever Pods are replaced or scaled. A Service gives a stable IP and DNS name and load-balances across the current Pods.
10. **Explain ClusterIP, NodePort and LoadBalancer.** Internal only / a port (30000–32767) on every node / a cloud load balancer in front of the NodePorts. Each one builds on the previous.
11. **What is a headless Service, and when would you use one?** `clusterIP: None`. DNS returns the individual Pod IPs, which is useful for databases and StatefulSets that need to reach specific Pods.
12. **What is an ExternalName Service?** A DNS alias (CNAME) to an outside hostname, with no selector and no proxying.
13. **Explain `containerPort`, `targetPort`, `port` and `nodePort`.** `nodePort` (on every node) → `port` (the Service) → `targetPort` (on the Pod), which must equal the port the app listens on (`containerPort`).
14. **A Service exists and the Pods are Running, but requests fail. What do you check?** `kubectl get endpoints <svc>`: if it's empty, the selector and labels don't match. If not, check that `targetPort` matches the port the app listens on.
15. **What does `kubectl port-forward` do?** It opens a temporary tunnel from your machine to a Pod or Service through the API server. It's for testing, not production.
16. **What is a DaemonSet? Give two use cases.** It runs exactly one Pod per node, and new nodes get one automatically. Examples: log collectors (Fluentd) and monitoring agents (node-exporter).
17. **What is a namespace?** A virtual partition of the cluster for separating teams or environments. Object names only need to be unique within a namespace.
