# Kubernetes Basics: Architecture & YAML

> **DevOps – Lecture 8 notes.** Anything marked **➕ Extra** was not covered in class. It's there because it fills a gap or comes up often in labs and interviews.
>
> **Builds on:** [Lecture 5–7 – Docker](05-Docker.md) (images, containers, networks, volumes). Kubernetes **runs** those containers for you across many machines.
>
> **Practice:** [Kubernetes MCQs](Kubernetes-MCQs.md) · **Next:** [Lecture 9 – Labels, Services, Deployments & DaemonSets](09-Kubernetes-Objects-Services-Deployments.md).

---

## Table of Contents
1. [The Big Picture: Why Kubernetes?](#1-the-big-picture-why-kubernetes)
2. [Key Terms](#2-key-terms)
3. [Architecture Overview](#3-architecture-overview)
4. [Control Plane (Master Node) Components](#4-control-plane-master-node-components)
5. [Worker Node Components](#5-worker-node-components)
6. [Controllers: Node & ReplicaSet](#6-controllers-node--replicaset)
7. [What Happens on `kubectl apply`](#7-what-happens-on-kubectl-apply)
8. [YAML Basics (just enough)](#8-yaml-basics-just-enough)
9. [The Four Top-Level Fields of a Kubernetes YAML](#9-the-four-top-level-fields-of-a-kubernetes-yaml)
10. [Hands-on: Your First Pod](#10-hands-on-your-first-pod)
11. [Pod Lifecycle](#11-pod-lifecycle)
12. [Hands-on: A Run-Once Task Pod](#12-hands-on-a-run-once-task-pod)
13. [➕ Preview: ReplicaSets & Deployments](#13--preview-replicasets--deployments)
14. [kubectl Command Reference](#14-kubectl-command-reference)
15. [Quick Revision Cheat Sheet](#15-quick-revision-cheat-sheet)
16. [Likely Exam / Interview Questions](#16-likely-exam--interview-questions)

---

## 1. The Big Picture: Why Kubernetes?

Docker runs containers on **one** machine. Real production apps need **dozens of containers across many servers**. Someone has to decide:
- Which server should each container run on?
- What if a container crashes, or a whole server dies?
- How do we scale from 3 copies to 30 during a sale?
- How do we update to a new version without downtime?

**Kubernetes (K8s)** is a **container orchestrator**: it does all of this automatically.

> 💡 **Simple words mein:** Docker = ek musician. Kubernetes = orchestra ka conductor. Kaun kab bajayega, koi gir gaya to replacement, sab conductor sambhalta hai.

**The core idea: desired state.** You don't tell K8s *how* to do things step by step. You write **what you want** in a YAML file ("3 copies of nginx, always"). Kubernetes **keeps checking reality against that** and fixes any difference, forever.

---

## 2. Key Terms

| Term | Meaning |
|------|---------|
| **Cluster** | A set of machines (nodes) managed together by Kubernetes |
| **Node** | One machine (VM or physical) in the cluster |
| **Control plane** (master node) | The "brain": makes decisions and stores the cluster state |
| **Worker node** | Where your app containers actually run |
| **Pod** | The **smallest unit** K8s deploys: a wrapper around **one (or a few) containers** that share an IP and storage |
| **`kubectl`** | The CLI you use to talk to the cluster (like `docker` for Docker) |
| **Manifest** | A YAML file describing an object you want |

> 💡 K8s never runs a container "bare". It always runs a **Pod**, and the Pod holds the container(s). One container per Pod is the usual case.

---

## 3. Architecture Overview

```
                    you ── kubectl ──▶
 ┌──────────────────── CONTROL PLANE (master node) ────────────────────┐
 │                                                                     │
 │   ┌────────────┐      ┌──────────────┐      ┌──────────────────┐    │
 │   │ API Server │◀────▶│     etcd     │      │    Scheduler     │    │
 │   │ (gatekeeper)│      │ (cluster DB) │      │ (picks a node)   │    │
 │   └─────▲──────┘      └──────────────┘      └────────▲─────────┘    │
 │         │      ┌───────────────────────────┐         │              │
 │         ├─────▶│    Controller Manager     │◀────────┘              │
 │         │      │ (node, replicaset, ...)   │   everyone talks       │
 │         │      └───────────────────────────┘   ONLY to the API      │
 └─────────┼───────────────────────────────────────────────────────────┘
           │
   ┌───────┴─────────────────┐            ┌─────────────────────────┐
   │      WORKER NODE 1       │            │      WORKER NODE 2       │
   │  kubelet   kube-proxy    │            │  kubelet   kube-proxy    │
   │  container runtime       │            │  container runtime       │
   │  [Pod] [Pod] [Pod]       │            │  [Pod] [Pod]             │
   └─────────────────────────┘            └─────────────────────────┘
```

> **Rule to remember:** **every** component talks **only to the API server**. Nothing else talks to etcd directly.

---

## 4. Control Plane (Master Node) Components

| Component | Role | Analogy |
|-----------|------|---------|
| **API Server** (`kube-apiserver`) | The **front door / gatekeeper**. Every request (from `kubectl`, the scheduler, kubelets) goes through it. It checks **who you are** and **whether you're allowed**, validates the YAML, then saves it to etcd. | Reception desk of an office |
| **etcd** | A key-value **database** holding **all cluster data and state**: every Pod, node, config, secret. | The office's record room |
| **Scheduler** (`kube-scheduler`) | Watches for new Pods with **no node assigned** and picks the best node, based on free CPU/memory and rules. It **only decides**. It doesn't start anything. | HR assigning a new joiner to a desk |
| **Controller Manager** (`kube-controller-manager`) | Runs many **control loops** (controllers) that keep the **actual state = desired state**. | Supervisors who keep checking "is everything as planned?" |
| ➕ Cloud Controller Manager | Talks to AWS/GCP/Azure: creates load balancers, attaches disks | Liaison with the building landlord |

> ⚠️ **Back up etcd.** If etcd is lost, the cluster forgets everything it was running.
>
> ➕ "Master node" is the older name. The official term today is **control plane**. Both mean the same thing in exams.

---

## 5. Worker Node Components

| Component | Role |
|-----------|------|
| **kubelet** | The **agent on every node**. It gets the list of Pods assigned to its node from the API server, tells the container runtime to **start/stop those containers**, keeps checking they're healthy, and **reports node and Pod status** back to the API server. |
| **kube-proxy** | Handles **networking** on the node. It sets up the rules (iptables) so traffic sent to a **Service** reaches the right Pods, even when Pods move around. |
| **Container runtime** ➕ | Actually runs the containers, e.g. **containerd** (what Docker itself uses underneath) or CRI-O. Images you built with `docker build` run here unchanged. |

> 💡 **Simple words mein:** kubelet = har node ka manager. Upar (API server) se order leta hai ("ye 3 pods chalao"), runtime se chalwata hai, aur report wapas bhejta hai.

---

## 6. Controllers: Node & ReplicaSet

A **controller** is a never-ending loop:
```
   ┌──▶ 1. read DESIRED state (from API server / etcd)
   │    2. look at ACTUAL state
   │    3. different? → take action to fix it
   └─── repeat forever
```
All the built-in controllers run inside the **Controller Manager**.

| Controller | Watches | Does |
|------------|---------|------|
| **Node controller** | Node heartbeats (sent by kubelets) | Marks a node **NotReady** when heartbeats stop (~40 s by default). After a grace period, its Pods are **evicted** and recreated on healthy nodes. |
| **ReplicaSet controller** | Number of running Pods with a given label | **Too few** (a Pod crashed or its node died) → creates more. **Too many** → deletes extras. Keeps exactly `replicas: N`. |
| ➕ Deployment, Job, … | — | Same loop idea for other object types |

> 💡 **Simple words mein:** AC ka thermostat. 24°C set kiya (desired), room 27°C hai (actual), to AC chalta rehta hai jab tak dono match na ho jaayein.

---

## 7. What Happens on `kubectl apply`

```
1. kubectl apply -f pod.yaml  ──▶  API Server: authenticate, authorize, validate YAML
2. API Server  ──▶  etcd: save Pod (status: Pending, node: <none>)
3. Scheduler sees an unassigned Pod  ──▶  picks node-2  ──▶  tells API Server
4. API Server  ──▶  etcd: Pod.node = node-2
5. kubelet on node-2 sees the new Pod  ──▶  container runtime pulls nginx image, starts it
6. kubelet  ──▶  API Server: "Pod is Running"  ──▶  etcd updated
7. kubectl get pods  →  nginx-pod   1/1   Running
```

This is a classic exam question. Note that **each step goes through the API server**.

---

## 8. YAML Basics (just enough)

YAML is a readable format for data. Kubernetes manifests are written in it.

```yaml
name: nginx            # key: value (space after the colon!)
replicas: 3            # number
enabled: true          # boolean
labels:                # nested object = indent with SPACES (2 is standard)
  app: web
  tier: frontend
ports:                 # list = items start with "- "
  - 80
  - 443
containers:            # list of objects
  - name: web
    image: nginx
```

> ⚠️ **Tabs are not allowed. Use spaces only.** Most "my YAML won't apply" errors are wrong indentation.
> ➕ `command: ["echo", "hi"]` (inline list) is the same as writing each item on its own `- ` line.

---

## 9. The Four Top-Level Fields of a Kubernetes YAML

**Every** Kubernetes object has these four:

```yaml
apiVersion: v1              # 1. which API version this object type belongs to
kind: Pod                   # 2. what TYPE of object
metadata:                   # 3. identity: name, labels, namespace
  name: nginx-pod
  labels:
    app: nginx
spec:                       # 4. the DESIRED STATE: what you want it to look like
  containers:
    - name: nginx-container
      image: nginx:latest
      ports:
        - containerPort: 80
```

| Field | What it holds | Examples |
|-------|---------------|----------|
| **apiVersion** | API group/version of the `kind` | `v1` (Pod, Service), `apps/v1` (Deployment, ReplicaSet) |
| **kind** | Object type | `Pod`, `ReplicaSet`, `Deployment`, `Service` |
| **metadata** | Name (must be unique per type in a namespace), labels | `name: nginx-pod` |
| **spec** | Configuration of the object | containers, image, ports, replicas |

➕ After creation K8s adds a fifth section, **`status`** (the **actual** state). You never write it yourself. See it with `kubectl get pod nginx-pod -o yaml`.

> ➕ Forgot the fields? `kubectl explain pod.spec.containers` prints the docs right in the terminal.

---

## 10. Hands-on: Your First Pod

**➕ Setup:** you need a cluster. Locally use **minikube** (`minikube start`), **kind**, or Docker Desktop's built-in Kubernetes. Check with `kubectl get nodes`.

**1. Save the YAML above as `pod.yaml`, then:**
```bash
kubectl apply -f pod.yaml           # create (or update) the Pod
kubectl get pods                    # NAME        READY   STATUS    RESTARTS   AGE
                                    # nginx-pod   1/1     Running   0          10s
kubectl get pods -o wide            # ➕ also shows the Pod's IP and which NODE it's on
kubectl describe pod nginx-pod      # full details + EVENTS (scheduled, pulled, started)
kubectl logs nginx-pod              # container's output
kubectl exec -it nginx-pod -- sh    # ➕ shell inside (like docker exec)
kubectl port-forward pod/nginx-pod 8080:80   # ➕ open http://localhost:8080 to test
kubectl delete pod nginx-pod        # delete (or: kubectl delete -f pod.yaml)
```

| Column | Meaning |
|--------|---------|
| `READY 1/1` | Ready containers / total containers in the Pod |
| `STATUS` | Phase or reason (see §11) |
| `RESTARTS` | How many times the container was restarted. A climbing number = crashing. |

> **Debug order:** `kubectl get pods` → `kubectl describe pod <name>` (look at **Events** at the bottom) → `kubectl logs <name>`.

---

## 11. Pod Lifecycle

```
 Pending ──▶ (ContainerCreating) ──▶ Running ──┬──▶ Succeeded (Completed)  all containers exited with 0
                                                └──▶ Failed                 a container exited with error
                                   Unknown = node stopped reporting
```

| Phase | Meaning | Class analogy (a task) |
|-------|---------|------------------------|
| **Pending** | Accepted, but not running yet: waiting for a node, or still pulling the image | Task assigned, not started |
| **Running** | Placed on a node, at least one container running | Task in progress |
| **Succeeded** (`Completed`) | All containers finished successfully and won't restart | Task done ✅ |
| **Failed** | Containers ended, at least one with an error | Task failed ❌ |
| **Unknown** | Node lost contact | Lost track of the task |

**➕ Common STATUS reasons you'll see in `kubectl get pods`:**
| STATUS | Usually means |
|--------|---------------|
| `ContainerCreating` | Pulling the image / starting (normal for a few seconds) |
| `ImagePullBackOff` / `ErrImagePull` | Wrong image name or tag, or a private registry without credentials |
| `CrashLoopBackOff` | The container keeps exiting and K8s keeps restarting it, waiting longer each time. Check `kubectl logs <pod> --previous`. |
| `Pending` (stuck) | No node has enough CPU/memory → see `describe` Events |

---

## 12. Hands-on: A Run-Once Task Pod

A Pod that runs **one command and finishes**, using the `command` field (it **overrides the image's default** `CMD`/`ENTRYPOINT`):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hello-pod
spec:
  restartPolicy: Never              # ⚠️ important, see below
  containers:
    - name: busybox-container
      image: busybox
      command: ["echo", "Hello Kubernetes"]
```

```bash
kubectl apply -f hello-pod.yaml
kubectl get pods                    # hello-pod   0/1   Completed   0   5s
kubectl logs hello-pod              # Hello Kubernetes
kubectl delete pod hello-pod
```

> ⚠️ **Why `restartPolicy: Never`?** The default is **`Always`**: K8s restarts a container every time it exits, even after success. Without this line, `echo` finishes, K8s restarts it, it finishes again… and you see **`CrashLoopBackOff`** with a climbing RESTARTS count instead of `Completed`.
>
> | `restartPolicy` | Restart when the container exits… |
> |-----------------|-------------------------------------|
> | `Always` (default) | always: for long-running servers like nginx |
> | `OnFailure` | only if the exit code ≠ 0 |
> | `Never` | never: one-off tasks |
>
> ➕ For real one-off tasks, use a **Job** object (`kind: Job`), which retries on failure and records completion.

**➕ `command` vs. `args`:** `command` replaces Docker's `ENTRYPOINT`, `args` replaces `CMD`.
```yaml
command: ["sh", "-c"]
args: ["echo Start; sleep 5; echo Done"]
```

---

## 13. ➕ Preview: ReplicaSets & Deployments

A bare Pod that dies **stays dead**. Nothing recreates it. In practice you almost never create Pods directly. You create a **Deployment**, which creates a **ReplicaSet**, which creates and watches the **Pods**.

```
 Deployment (rolling updates, rollbacks)
     └── ReplicaSet (keeps N copies alive)   ← the ReplicaSet controller from §6
            ├── Pod
            ├── Pod
            └── Pod
```

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx              # "the Pods I own are the ones labelled app=nginx"
  template:                   # the Pod blueprint, i.e. a Pod's metadata + spec
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
```

**See self-healing:**
```bash
kubectl apply -f deploy.yaml
kubectl get pods                      # 3 nginx-deploy-xxxx Pods
kubectl delete pod <one-of-them>
kubectl get pods                      # still 3: a new one was created instantly ✅
kubectl scale deployment nginx-deploy --replicas=5
```

---

## 14. kubectl Command Reference

| Task | Command |
|------|---------|
| Create / update from YAML | `kubectl apply -f file.yaml` |
| List Pods | `kubectl get pods` (`-o wide`, `-A` all namespaces, `-w` watch) |
| List anything | `kubectl get nodes` / `deployments` / `rs` / `all` |
| Details + events | `kubectl describe pod <name>` |
| Logs | `kubectl logs <pod>` (`-f` follow, `--previous` last crashed run) |
| Shell inside | `kubectl exec -it <pod> -- sh` |
| Delete | `kubectl delete pod <name>` / `kubectl delete -f file.yaml` |
| See full live YAML | `kubectl get pod <name> -o yaml` |
| ➕ Quick Pod without YAML | `kubectl run nginx --image=nginx` |
| ➕ Generate YAML for you | `kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml` |
| ➕ Docs in the terminal | `kubectl explain pod.spec` |

---

## 15. Quick Revision Cheat Sheet

```
WHY K8S        orchestrate containers across many nodes: place, heal, scale, update · DESIRED STATE
CLUSTER        control plane (brain) + worker nodes (run Pods)       POD = smallest unit, wraps container(s)
CONTROL PLANE  API Server  = gatekeeper, only entry point, authn/authz/validate
               etcd        = key-value DB of ALL cluster state (back it up!)
               Scheduler   = picks a node for new Pods (decides only)
               Controller Mgr = control loops: actual → desired (node, replicaset, deployment, job)
WORKER         kubelet = node agent: runs assigned Pods via runtime, reports status
               kube-proxy = Service networking rules       runtime = containerd / CRI-O
APPLY FLOW     kubectl → API → etcd → scheduler → API → kubelet → runtime → status → API → etcd
NODE CTRL      no heartbeat → NotReady → evict Pods      RS CTRL = keep exactly N Pods
YAML           spaces not tabs · key: value · "- " list items
4 FIELDS       apiVersion (v1 / apps/v1) · kind · metadata (name, labels) · spec (desired)   + status (actual)
POD PHASES     Pending → Running → Succeeded (Completed) | Failed · Unknown
ERRORS         ImagePullBackOff = bad image · CrashLoopBackOff = keeps exiting · Pending = no resources
RUN-ONCE       command: [...] + restartPolicy: Never (default Always → CrashLoopBackOff)
KUBECTL        apply -f · get pods [-o wide] · describe · logs [--previous] · exec -it -- sh · delete
```

---

## 16. Likely Exam / Interview Questions

1. **What is Kubernetes and why do we need it when we have Docker?** Docker runs containers on one host. Kubernetes orchestrates them across many hosts: scheduling, self-healing, scaling and rolling updates.
2. **What does "desired state" mean?** You declare what you want in YAML, and controllers continuously make the actual state match it.
3. **Name the control plane components and their roles.** API server (gatekeeper), etcd (state store), scheduler (picks nodes), controller manager (control loops).
4. **Why is the API server called the gatekeeper?** It's the only entry point. It authenticates, authorizes and validates every request, and it's the only component that talks to etcd.
5. **What is stored in etcd?** All cluster data: objects, their desired and actual state, config and secrets.
6. **Does the scheduler start containers?** No. It only assigns a node. The kubelet on that node starts the containers.
7. **What does the kubelet do?** It runs on every worker node, starts the Pods assigned to it via the container runtime, monitors them, and reports status to the API server.
8. **What does kube-proxy do?** It maintains the networking rules on each node so Service traffic reaches the right Pods.
9. **Explain the node controller and the ReplicaSet controller.** The node controller detects nodes that stop sending heartbeats and evicts their Pods. The ReplicaSet controller keeps the configured number of Pod replicas running.
10. **Walk through what happens when you run `kubectl apply -f pod.yaml`.** See [§7](#7-what-happens-on-kubectl-apply).
11. **What are the four required top-level fields in a manifest?** `apiVersion`, `kind`, `metadata`, `spec`.
12. **`spec` vs. `status`?** `spec` is the desired state you write. `status` is the actual state that Kubernetes reports.
13. **What are the Pod phases?** Pending, Running, Succeeded, Failed, Unknown.
14. **A Pod that runs `echo` shows `CrashLoopBackOff`. Why, and how do you fix it?** The default `restartPolicy: Always` keeps restarting the finished container. Set `restartPolicy: Never` (or `OnFailure`), or use a Job.
15. **Pod vs. Deployment?** A bare Pod isn't recreated if it dies. A Deployment (through a ReplicaSet) keeps N replicas alive and handles rolling updates.
16. **How do you debug a Pod stuck in `ImagePullBackOff`?** `kubectl describe pod <name>` and read the Events. Usually it's a wrong image name or tag, or missing registry credentials.
