# Kubernetes Part 3: Deployment Strategies & Pod Lifecycle

> **DevOps – Lecture 10 notes.** Anything marked **➕ Extra** was not covered in class. It's there because it fills a gap or comes up often in labs and interviews.
>
> **Builds on:** [Lecture 8 – Kubernetes Basics](08-Kubernetes-Basics.md) (Pods, lifecycle, YAML) and [Lecture 9 – Labels, Services, Deployments & DaemonSets](09-Kubernetes-Objects-Services-Deployments.md) (selectors, Services, `maxSurge` / `maxUnavailable`). Topics already covered there are only **recapped and linked** here, not repeated.
>
> **Practice:** [Kubernetes MCQs](Kubernetes-MCQs.md) · **Next:** [Lecture 11 – ConfigMaps, Secrets & Ingress](11-Kubernetes-ConfigMaps-Secrets-Ingress.md).

---

## Table of Contents
1. [Quick Recap](#1-quick-recap)
2. [Pod Lifecycle & Debugging Toolkit](#2-pod-lifecycle--debugging-toolkit)
3. [Why Deployment Strategies Matter](#3-why-deployment-strategies-matter)
4. [Strategy 1: Rolling Update](#4-strategy-1-rolling-update)
5. [Strategy 2: Recreate](#5-strategy-2-recreate)
6. [Strategy 3: Blue-Green](#6-strategy-3-blue-green)
7. [Strategy 4: Canary](#7-strategy-4-canary)
8. [Comparing the Four Strategies](#8-comparing-the-four-strategies)
9. [Worked Example: Rolling Update → Verify → Roll Back](#9-worked-example-rolling-update--verify--roll-back)
10. [DaemonSets Recap](#10-daemonsets-recap)
11. [Quick Revision Cheat Sheet](#11-quick-revision-cheat-sheet)
12. [Likely Exam / Interview Questions](#12-likely-exam--interview-questions)

---

## 1. Quick Recap

| Concept | One line | More in |
|---------|----------|---------|
| `kubectl get` | Lists objects: `get pods`, `get nodes`, `get deploy`, `get rs` | [L9 §2](09-Kubernetes-Objects-Services-Deployments.md#2-everyday-kubectl-commands) |
| `kubectl describe` | Full details + **Events** for one object | [L9 §2](09-Kubernetes-Objects-Services-Deployments.md#2-everyday-kubectl-commands) |
| **Pod** | Smallest deployable unit, wraps container(s) | [L8 §2](08-Kubernetes-Basics.md#2-key-terms) |
| **ReplicaSet** | Keeps N identical Pods running | [L9 §5](09-Kubernetes-Objects-Services-Deployments.md#5-replicasets) |
| **Deployment** | Manages ReplicaSets, so you get updates + rollback | [L9 §6](09-Kubernetes-Objects-Services-Deployments.md#6-deployments) |
| **DaemonSet** | One Pod on every node | [L9 §11](09-Kubernetes-Objects-Services-Deployments.md#11-daemonsets) |
| **Service + selector** | Stable address that sends traffic to Pods with matching labels | [L9 §8](09-Kubernetes-Objects-Services-Deployments.md#8-services-why-we-need-them) |

> The last row is the key to this lecture: **blue-green and canary are built by playing with labels and Service selectors.**

---

## 2. Pod Lifecycle & Debugging Toolkit

### The lifecycle in one line
```
 Pending ──▶ Running ──▶ Succeeded / Failed          (Unknown = node lost contact)
```
Full table and common STATUS reasons (`ImagePullBackOff`, `CrashLoopBackOff`, …): [L8 §11](08-Kubernetes-Basics.md#11-pod-lifecycle).

### Pod YAML: see what K8s actually stored
```bash
kubectl get pod <pod> -o yaml       # your spec + K8s defaults + live "status" section
kubectl get deploy my-app -o yaml   # same for any object
```

### Logs: understand what the app is doing
```bash
kubectl logs <pod>                  # output of the (only) container
kubectl logs -f <pod>               # follow live, like tail -f
kubectl logs <pod> --previous       # the run that CRASHED (for CrashLoopBackOff)
kubectl logs <pod> -c <container>   # ➕ pick one container in a multi-container Pod
kubectl logs -l app=my-app          # ➕ logs from every Pod with that label
kubectl logs deploy/my-app          # ➕ logs from one Pod of the Deployment
```

### Debug order (memorise this)
```
 kubectl get pods  ──▶  STATUS / RESTARTS look wrong?
        │
        ▼
 kubectl describe pod <pod>  ──▶  read EVENTS at the bottom (scheduling, image pull, probe failures)
        │
        ▼
 kubectl logs <pod> [--previous]  ──▶  the app's own error message
```

### ➕ Probes: how K8s knows a Pod is healthy
Rollouts rely on these, so they belong here.

| Probe | Question it answers | On failure |
|-------|---------------------|------------|
| **readinessProbe** | "Can this Pod **receive traffic** yet?" | Removed from the Service endpoints (not restarted). A rolling update **waits** for it. |
| **livenessProbe** | "Is this Pod **stuck / dead**?" | Container is **restarted** |

```yaml
containers:
  - name: web
    image: my-app:2.0
    readinessProbe:
      httpGet: { path: /health, port: 8080 }
      initialDelaySeconds: 5
      periodSeconds: 5
    livenessProbe:
      httpGet: { path: /health, port: 8080 }
      initialDelaySeconds: 15
```
> 💡 **Simple words mein:** readiness = "kya dukaan khul gayi, customer bhej sakte hain?" Liveness = "kya dukaandaar so to nahi gaya? So gaya to uthao (restart)."

---

## 3. Why Deployment Strategies Matter

Shipping a new version is the **riskiest moment** in production: new bugs, bad config, slow startup. A **deployment strategy** decides:
- Will users see **downtime**?
- Will **old and new versions run together** for a while?
- How **many users** are hit if the new version is broken?
- How **fast** can we go back?
- How many **extra resources** do we need during the switch?

| Built into Kubernetes (`spec.strategy.type`) | Built **by you** with labels + Services |
|----------------------------------------------|-----------------------------------------|
| **RollingUpdate** (default), **Recreate** | **Blue-Green**, **Canary** |

---

## 4. Strategy 1: Rolling Update

**Gradually** replaces old Pods with new ones, a few at a time, so the app **never goes down**. It's the **default** strategy.

> 💡 **Analogy (from class): a conveyor belt.** You swap one old product for a new one while the belt keeps moving. The factory never stops.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  replicas: 3
  selector:                    # required (missing in the class handout)
    matchLabels:
      app: my-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1              # up to 3 + 1 = 4 Pods in total
      maxUnavailable: 1        # at least 3 − 1 = 2 Pods available
  template:
    metadata:
      labels:
        app: my-app            # must match the selector
    spec:
      containers:
        - name: web
          image: my-app:1.0
```

### Walk-through: `replicas: 3`, `maxSurge: 1`, `maxUnavailable: 1`
```
 Limits: max 4 Pods total, min 2 available

 Step   Old (v1)   New (v2)          Available   What happened
  0       3          0                  3        start
  1       2          1 (starting)       2        +1 new (surge) and −1 old (unavailable) AT THE SAME TIME
  2       1          2 (1 starting)     2        first new Pod Ready → one more swap
  3       0          3                  3        done ✅
```
This is **faster** than `maxSurge: 1, maxUnavailable: 0` (the step table in [L9 §7](09-Kubernetes-Objects-Services-Deployments.md#7-rolling-updates-maxsurge--maxunavailable)), because it adds and removes in the same step. The cost is running with **one fewer Pod** for a short time.

| ✅ Pros | ❌ Cons |
|--------|--------|
| Zero downtime, built in | **v1 and v2 serve users together** for a while, so they must be compatible (same DB schema, same API) |
| Few extra resources (just `maxSurge`) | Rollback is another rolling update, not instant |
| Easy rollback: `kubectl rollout undo` | Every user gets v2 eventually, so you can't target a small group |

---

## 5. Strategy 2: Recreate

**Kill all old Pods first, then start the new ones.**

```yaml
spec:
  replicas: 3
  strategy:
    type: Recreate            # no rollingUpdate block
```
```
 v1 v1 v1  ──▶  (nothing running: DOWNTIME ⚠️)  ──▶  v2 v2 v2
```

| ✅ Pros | ❌ Cons |
|--------|--------|
| Simplest, and the switch itself is quick | **Downtime** while new Pods start |
| **Old and new never run together** | Rollback means more downtime |
| No extra resources needed | |

**Use when:**
- Downtime is acceptable (dev/test, internal tools, a maintenance window).
- The two versions **can't coexist**: an incompatible DB schema change, or the app needs exclusive access to a volume or license.

---

## 6. Strategy 3: Blue-Green

Run **two full environments**:
- **Blue** = the current version, serving **all** traffic.
- **Green** = the new version, deployed **alongside** it and tested **while users still use blue**.

When green is verified, **switch all traffic at once**. If something breaks, **switch back instantly**.

```
 BEFORE                                   AFTER the switch
 Users ──▶ Service (version: blue)        Users ──▶ Service (version: green)
              │                                         │
              ▼                                         ▼
     [blue v1] [blue v1] [blue v1]           [green v2] [green v2] [green v2]
     [green v2] [green v2] [green v2]        [blue v1] [blue v1] [blue v1]  ← kept idle for instant rollback
       (deployed, tested, no users)
```

> 💡 **Simple words mein:** do stage taiyaar hain. Blue stage pe show chal raha hai, green stage pe rehearsal. Rehearsal theek lagi to audience ko ek saath green stage ki taraf ghuma do. Gadbad hui to wapas blue.

### How to build it in Kubernetes (two Deployments + one Service)
```yaml
# blue.yaml: the current version
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-blue
spec:
  replicas: 3
  selector:
    matchLabels: { app: my-app, version: blue }
  template:
    metadata:
      labels: { app: my-app, version: blue }
    spec:
      containers:
        - name: web
          image: my-app:1.0
---
# green.yaml: identical except name, version label and image
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-green
spec:
  replicas: 3
  selector:
    matchLabels: { app: my-app, version: green }
  template:
    metadata:
      labels: { app: my-app, version: green }
    spec:
      containers:
        - name: web
          image: my-app:2.0
---
# service.yaml: the "traffic switch"
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app: my-app
    version: blue            # ← flip this to "green" to switch ALL traffic
  ports:
    - port: 80
      targetPort: 8080
```

```bash
kubectl apply -f blue.yaml -f service.yaml           # live on blue
kubectl apply -f green.yaml                          # deploy green next to it (no traffic yet)
kubectl port-forward deploy/my-app-green 8080:8080   # test green privately

# SWITCH: one command, instant
kubectl patch svc my-app -p '{"spec":{"selector":{"app":"my-app","version":"green"}}}'

# ROLLBACK: just as instant
kubectl patch svc my-app -p '{"spec":{"selector":{"app":"my-app","version":"blue"}}}'

# Once green has been stable for a while, remove blue
kubectl delete deploy my-app-blue
```

| ✅ Pros | ❌ Cons |
|--------|--------|
| **Zero downtime** and an **instant switch** | **Double the resources** during the release |
| **Instant rollback**: flip the selector back | All users move at once, so a hidden bug hits **everyone** |
| Test the new version in real production conditions before users see it | Database changes must work for both versions |

---

## 7. Strategy 4: Canary

Release the new version to a **small slice of users first** (say 10%). Watch errors and latency. If it's healthy, increase step by step (10% → 25% → 50% → 100%). If not, remove it, and **only a few users were affected**.

> 💡 **Name:** miners took a **canary bird** into the mine. If there was gas, the canary reacted first and warned everyone. The canary Pods are your early warning.
>
> 💡 **Simple words mein:** naya dish pehle sirf 10% customers ko do. Sab khush → sabko do. Shikayat aayi → menu se hata do, baaki 90% ko pata bhi nahi chala.

### How to build it in Kubernetes (basic version: ratio by Pod count)
The Service selects **only `app: my-app`**, so it load-balances across **both** Deployments. The traffic split follows the **number of Pods**.

```yaml
# stable: 9 Pods of v1
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-stable
spec:
  replicas: 9
  selector:
    matchLabels: { app: my-app, track: stable }
  template:
    metadata:
      labels: { app: my-app, track: stable }
    spec:
      containers:
        - name: web
          image: my-app:1.0
---
# canary: 1 Pod of v2  →  1 / (9 + 1) ≈ 10% of traffic
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-canary
spec:
  replicas: 1
  selector:
    matchLabels: { app: my-app, track: canary }
  template:
    metadata:
      labels: { app: my-app, track: canary }
    spec:
      containers:
        - name: web
          image: my-app:2.0
---
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  selector:
    app: my-app              # NO track label → matches stable AND canary Pods
  ports:
    - port: 80
      targetPort: 8080
```

```bash
# Watch the canary closely
kubectl logs -f -l track=canary

# Healthy → shift more traffic to it
kubectl scale deploy my-app-canary --replicas=3     # 3/12 = 25%
kubectl scale deploy my-app-stable --replicas=6     # 3/9  ≈ 33%

# Fully healthy → move stable to v2 and delete the canary
kubectl set image deploy/my-app-stable web=my-app:2.0
kubectl delete deploy my-app-canary

# Broken → remove it, and 100% of traffic goes back to stable
kubectl delete deploy my-app-canary   # or: kubectl scale deploy my-app-canary --replicas=0
```

> ⚠️ **Limitation:** with plain Services the split is **tied to Pod counts** (1% would need 99 stable Pods), and a given user may hit v1 and v2 on different requests.
> ➕ **Precise splits** ("exactly 5%", or "only users with header `beta=true`") come from an **Ingress controller** (NGINX canary annotations), a **service mesh** (Istio, Linkerd) or **Argo Rollouts** / **Flagger**, which also automate the "watch metrics → promote / abort" loop.

| ✅ Pros | ❌ Cons |
|--------|--------|
| **Smallest blast radius**: a bug hits only a few users | Slower: several stages plus watching between them |
| Tested with **real users and real traffic** | Needs **good monitoring** to judge the canary |
| Easy abort: delete the canary | Two versions run together, so they must be compatible |

### ➕ Blue-green vs. canary: the one-line difference
- **Blue-green** moves **100% of traffic at once**, between two full environments.
- **Canary** moves traffic **gradually**, from a small group to everyone.

---

## 8. Comparing the Four Strategies

| | Rolling Update | Recreate | Blue-Green | Canary |
|-|----------------|----------|------------|--------|
| **Downtime** | None | **Yes** | None | None |
| **Old + new run together?** | Yes, briefly | **Never** | Yes, but only one gets traffic | Yes, both get traffic |
| **Users exposed to a bad release** | Gradually all | All | All (at the switch) | **Only the canary %** |
| **Rollback speed** | Medium (`rollout undo`) | Slow (downtime again) | **Instant** (flip selector) | **Fast** (delete canary) |
| **Extra resources** | Low (`maxSurge`) | None | **High (2×)** | Low |
| **Built into K8s?** | ✅ Default | ✅ `type: Recreate` | ❌ Labels + Service | ❌ Labels + Service (or tools) |
| **Best for** | Most everyday releases | Dev/test, incompatible versions | Big releases needing an instant rollback | Risky changes, large user bases |

**Choosing quickly:**
```
 Can the two versions run at the same time?
   NO  ──▶ Recreate
   YES ──▶ Need an instant, all-at-once switch with an instant rollback?
             YES ──▶ Blue-Green (if you can afford 2× resources)
             NO  ──▶ Want to test on a small % of real users first?
                       YES ──▶ Canary
                       NO  ──▶ Rolling Update (default)
```

---

## 9. Worked Example: Rolling Update → Verify → Roll Back

**1. Deploy v1** (YAML from [§4](#4-strategy-1-rolling-update), image `nginx:1.26`, saved as `my-app.yaml`)
```bash
kubectl apply -f my-app.yaml
kubectl annotate deploy my-app kubernetes.io/change-cause="v1: nginx 1.26"   # ➕ label this revision
kubectl get pods -l app=my-app
```

**2. Update to v2:** change `image:` to `nginx:1.27` in the YAML, then apply again
```bash
kubectl apply -f my-app.yaml          # or: kubectl set image deploy/my-app web=nginx:1.27
kubectl annotate deploy my-app kubernetes.io/change-cause="v2: nginx 1.27" --overwrite
```

**3. Verify**
```bash
kubectl rollout status deploy/my-app  # Waiting for rollout... → "successfully rolled out"
kubectl get pods -w                   # ➕ watch old Pods terminate, new ones start
kubectl get rs                        # old RS → 0, new RS → 3
kubectl describe deploy my-app        # Events: "Scaled up replica set ... Scaled down ..."
```

**4. Something's wrong → roll back**
```bash
kubectl rollout history deploy/my-app
# REVISION  CHANGE-CAUSE
# 1         v1: nginx 1.26
# 2         v2: nginx 1.27
kubectl rollout undo deploy/my-app                  # back to revision 1
kubectl rollout undo deploy/my-app --to-revision=1  # or name the exact revision
```

**➕ Useful extras**
```bash
kubectl rollout pause deploy/my-app    # freeze mid-rollout (a "manual canary": some old, some new)
kubectl rollout resume deploy/my-app   # continue
kubectl rollout restart deploy/my-app  # re-create all Pods with the SAME version (e.g. to reload config)
```
| ➕ Field | Meaning |
|---------|---------|
| `revisionHistoryLimit: 10` | How many old ReplicaSets to keep for rollback (default 10) |
| `progressDeadlineSeconds: 600` | If the rollout makes no progress for this long, it's marked **failed** (e.g. new Pods never become Ready) |

> ⚠️ A broken image (`ImagePullBackOff`) with `maxUnavailable: 0` **won't take the app down**. The new Pods never become Ready, so the old ones are never removed. The rollout just stalls. That's rolling updates protecting you.

---

## 10. DaemonSets Recap

**One Pod on every node**, added automatically when a node joins. Used for **log collectors** (Fluentd, Fluent Bit) and **monitoring agents** (node-exporter). Full notes and YAML: [L9 §11](09-Kubernetes-Objects-Services-Deployments.md#11-daemonsets).

**➕ Two things worth knowing:**
```yaml
spec:
  updateStrategy:
    type: RollingUpdate          # DaemonSets also roll out node by node (default)
    rollingUpdate:
      maxUnavailable: 1          # update 1 node's Pod at a time
  template:
    spec:
      nodeSelector:              # run only on SOME nodes, e.g. those labelled disk=ssd
        disk: ssd
```
- A DaemonSet has **no `maxSurge` in practice** (only one Pod is allowed per node), so updates replace the Pod on each node in turn.
- `nodeSelector` limits it to a subset of nodes. Without it, every node gets one.

---

## 11. Quick Revision Cheat Sheet

```
RECAP         get (list) · describe (details + Events) · Pod < ReplicaSet < Deployment · DaemonSet = 1/node
LIFECYCLE     Pending → Running → Succeeded | Failed · Unknown
DEBUG         get pods → describe pod (Events) → logs [--previous] [-f] [-c ctr] [-l label]
              get pod x -o yaml = full stored spec + status
PROBES ➕     readiness = ready for traffic (out of Service if failing) · liveness = alive (restart if failing)
ROLLING       default · gradual · zero downtime · v1+v2 together · maxSurge (extra) / maxUnavailable (missing)
              3 replicas, surge 1, unavail 1 → max 4 Pods, min 2 available · conveyor belt
RECREATE      kill all → start all · DOWNTIME · versions never coexist · type: Recreate
BLUE-GREEN    2 full Deployments (version: blue / green) · Service selector = switch
              kubectl patch svc ... version: green · instant switch + instant rollback · 2× resources
CANARY        small % first → watch → increase · Service selects app only, split ≈ Pod ratio (1 of 10 = 10%)
              abort = delete canary · precise % → Ingress / Istio / Argo Rollouts ➕
CHOOSE        can't coexist → Recreate · instant flip → Blue-Green · small % test → Canary · else Rolling
ROLLOUT       apply (new tag) · rollout status · rollout history · rollout undo [--to-revision=N]
              pause / resume / restart ➕ · change-cause annotation ➕
DAEMONSET     logs / monitoring agents · updateStrategy RollingUpdate · nodeSelector for a subset ➕
```

---

## 12. Likely Exam / Interview Questions

1. **What deployment strategies does Kubernetes support out of the box?** `RollingUpdate` (the default) and `Recreate`. Blue-green and canary are built with labels and Services, or with tools.
2. **Explain a rolling update and its two key parameters.** Pods are replaced gradually. `maxSurge` is how many extra Pods may exist above `replicas`. `maxUnavailable` is how many may be missing below it.
3. **With 3 replicas, `maxSurge: 1` and `maxUnavailable: 1`, what are the limits during an update?** At most 4 Pods in total, and at least 2 available.
4. **When would you choose Recreate over a rolling update?** When the two versions can't run together (incompatible schema, exclusive resource) and downtime is acceptable.
5. **What is a blue-green deployment, and how do you implement it in Kubernetes?** Two full Deployments labelled `version: blue` and `version: green`. The Service selector points at one of them. Change the selector to switch traffic instantly.
6. **How do you roll back a blue-green deployment?** Point the Service selector back at blue. It's instant, because blue is still running.
7. **What is the main drawback of blue-green?** It needs double the resources during the release, and all users switch at once.
8. **What is a canary deployment, and why use it?** Expose the new version to a small percentage of users first, so a bad release only affects a few people.
9. **How do you send ~10% of traffic to a canary using only Deployments and a Service?** Run 9 stable Pods and 1 canary Pod with a shared `app` label, and have the Service select only `app`.
10. **Blue-green vs. canary?** Blue-green switches 100% of traffic at once between two environments. Canary shifts traffic gradually.
11. **Which strategies run two versions at the same time? Why does that matter?** Rolling update and canary, plus blue-green while both environments exist. Both versions must then be compatible with the same database and APIs.
12. **How do you check that a rollout succeeded, and how do you undo it?** `kubectl rollout status`, `kubectl get pods`, `kubectl get rs`. Undo with `kubectl rollout undo [--to-revision=N]`.
13. **A new image has a typo in its tag, and `maxUnavailable` is 0. Does the app go down?** No. The new Pods stay in `ImagePullBackOff` and never become Ready, so the old Pods are never removed and the rollout stalls.
14. **What's the difference between a readiness probe and a liveness probe?** Failing readiness removes the Pod from the Service, without a restart. Failing liveness restarts the container.
15. **How do you get the logs of a container that crashed and restarted?** `kubectl logs <pod> --previous`.
16. **What is a DaemonSet, and how does it differ from a Deployment?** It runs one Pod per node, with no `replicas` field, and grows as nodes join. It's used for node-level agents like log and monitoring collectors.
