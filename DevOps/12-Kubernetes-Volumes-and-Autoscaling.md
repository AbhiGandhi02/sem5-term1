# Kubernetes Part 5: Volumes, Persistent Storage & Autoscaling (HPA)

> **DevOps – Lecture 12 notes.** Anything marked **➕ Extra** was not covered in class. It's there because it fills a gap or comes up often in labs and interviews.
>
> **Builds on:** [Lecture 7 – Docker Volumes](07-Docker-Networking-and-Volumes.md#7-docker-storage-why-volumes) (same idea, now cluster-wide), [Lecture 9 – Services & Deployments](09-Kubernetes-Objects-Services-Deployments.md) and [Lecture 11 – ConfigMaps & Secrets](11-Kubernetes-ConfigMaps-Secrets-Ingress.md). Topics already covered there are only **recapped and linked** here.
>
> **Practice:** [Kubernetes MCQs](Kubernetes-MCQs.md) · **Next:** [Lecture 13 – Troubleshooting & Helm](13-Kubernetes-Troubleshooting-and-Helm.md).

---

## Table of Contents
1. [Recap: ConfigMaps, Secrets & Service Types](#1-recap-configmaps-secrets--service-types)
2. [Why Pods Need Volumes](#2-why-pods-need-volumes)
3. [`emptyDir`: Scratch Space for a Pod](#3-emptydir-scratch-space-for-a-pod)
4. [`hostPath`: A Folder from the Node](#4-hostpath-a-folder-from-the-node)
5. [Persistent Volumes (PV) & Persistent Volume Claims (PVC)](#5-persistent-volumes-pv--persistent-volume-claims-pvc)
6. [Static Provisioning: Write the PV Yourself](#6-static-provisioning-write-the-pv-yourself)
7. [Dynamic Provisioning: StorageClass](#7-dynamic-provisioning-storageclass)
8. [Access Modes, Reclaim Policies & PV Phases](#8-access-modes-reclaim-policies--pv-phases)
9. [Comparing the Volume Types](#9-comparing-the-volume-types)
10. [Horizontal Pod Autoscaler (HPA)](#10-horizontal-pod-autoscaler-hpa)
11. [Hands-on Lab: Watch HPA Scale Under Load](#11-hands-on-lab-watch-hpa-scale-under-load)
12. [➕ HPA vs. VPA vs. Cluster Autoscaler](#12--hpa-vs-vpa-vs-cluster-autoscaler)
13. [Quick Revision Cheat Sheet](#13-quick-revision-cheat-sheet)
14. [Likely Exam / Interview Questions](#14-likely-exam--interview-questions)

---

## 1. Recap: ConfigMaps, Secrets & Service Types

| Topic | One line | Full notes |
|-------|----------|------------|
| **ConfigMap** | Non-sensitive, environment-specific config (app properties, log level) | [L11 §3–4](11-Kubernetes-ConfigMaps-Secrets-Ingress.md#3-configmaps) |
| **Secret** | Sensitive data (passwords, OAuth tokens), stored **base64-encoded**, which is **not encryption** | [L11 §5–6](11-Kubernetes-ConfigMaps-Secrets-Ingress.md#5-secrets) |
| **Service types** | ClusterIP (internal, default) · NodePort (port on every node) · LoadBalancer (cloud LB) · ExternalName (DNS alias) · Headless (`clusterIP: None`, direct Pod IPs) | [L9 §9](09-Kubernetes-Objects-Services-Deployments.md#9-service-types) |

**Commands from class:**
```bash
kubectl apply -f configmap.yaml          kubectl apply -f secret.yaml
kubectl get configmaps                   kubectl get secrets
kubectl describe configmap <name>        kubectl describe secret <name>     # values hidden, sizes only
kubectl delete configmap <name>          kubectl delete secret <name>
```

> ➕ ConfigMaps and Secrets are themselves a **kind of volume**: mount them as files with `volumes: - configMap:` / `- secret:` ([L11 §4](11-Kubernetes-ConfigMaps-Secrets-Ingress.md#4-using-a-configmap-in-a-pod)). This lecture covers the volumes that hold **data**.

---

## 2. Why Pods Need Volumes

A container's filesystem is **temporary**, exactly as in Docker ([L7 §7](07-Docker-Networking-and-Volumes.md#7-docker-storage-why-volumes)):
- Container **crashes and restarts** → its files are **reset** to the image.
- Pod is **deleted / rescheduled** to another node → everything is **gone**.
- Two containers in one Pod **can't see each other's files**.

A **volume** is a directory mounted into the Pod that solves these problems. Each type keeps data for a different length of time:

```
 Lifetime of the data:
 container fs   ── until container restarts
 emptyDir       ── until the POD is deleted
 hostPath       ── as long as that NODE's disk exists
 PV / PVC       ── independent of Pods and nodes (real persistence) ✅
```

Every volume is used in two steps:
```yaml
spec:
  volumes:                  # 1. DECLARE the volume at Pod level
    - name: data
      <type>: {...}
  containers:
    - name: app
      volumeMounts:         # 2. MOUNT it into a container at a path
        - name: data
          mountPath: /data
```

---

## 3. `emptyDir`: Scratch Space for a Pod

An **empty folder created when the Pod starts** and **deleted when the Pod is removed**.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: emptydir-demo
spec:
  containers:
    - name: writer
      image: busybox
      command: ["sh", "-c", "while true; do date >> /cache/log.txt; sleep 5; done"]
      volumeMounts:
        - name: cache
          mountPath: /cache
    - name: reader                       # a 2nd container sees the SAME folder
      image: busybox
      command: ["sh", "-c", "tail -f /cache/log.txt"]
      volumeMounts:
        - name: cache
          mountPath: /cache
  volumes:
    - name: cache
      emptyDir: {}                       # ➕ emptyDir: { medium: Memory } = RAM (tmpfs), faster
```
```bash
kubectl logs emptydir-demo -c reader     # dates written by the "writer" container
```

| Survives… | ? |
|-----------|---|
| Container crash / restart | ✅ Yes (the Pod is still alive) |
| Pod deleted, or rescheduled to another node | ❌ No |

**Use for:** caches, temp files, scratch space for processing, and **sharing files between containers in the same Pod** (e.g. an app writes logs, a sidecar ships them).

> 💡 **Simple words mein:** emptyDir = exam hall ki rough sheet. Exam (Pod) chalne tak use karo, jitne students (containers) us bench pe hain sab dekh sakte hain. Exam khatam, sheet koode mein.

---

## 4. `hostPath`: A Folder from the Node

Mounts a **file or directory from the node's own filesystem** into the Pod.

```yaml
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "echo hello >> /data/out.txt; sleep 3600"]
      volumeMounts:
        - name: host-data
          mountPath: /data
  volumes:
    - name: host-data
      hostPath:
        path: /mnt/data              # folder on the NODE
        type: DirectoryOrCreate      # ➕ create it if it doesn't exist
```

| Survives… | ? |
|-----------|---|
| Container restart | ✅ |
| Pod deleted and recreated **on the same node** | ✅ |
| Pod rescheduled to a **different node** | ❌ **The data stays behind** on the old node |

> ⚠️ **Why hostPath is risky in real clusters:**
> - **Tied to one node.** Pods can move, the data can't.
> - **Security:** a Pod can read or modify the node's files (`hostPath: /` = full access to the host).
>
> **Use for:** single-node labs (minikube), and **node-level agents** that genuinely need host files, e.g. a log-collector DaemonSet reading `/var/log` ([L9 §11](09-Kubernetes-Objects-Services-Deployments.md#11-daemonsets)). **Not for app data in production.**

---

## 5. Persistent Volumes (PV) & Persistent Volume Claims (PVC)

For **real persistence** (databases, uploads), storage must live **outside Pods and nodes**: a cloud disk (AWS EBS), a network file share (NFS, EFS) and so on. Kubernetes splits this into two objects:

| | **PersistentVolume (PV)** | **PersistentVolumeClaim (PVC)** |
|-|---------------------------|---------------------------------|
| What | A **piece of actual storage** in the cluster | A **request** for storage by an app |
| Says | "I am 1 Gi, ReadWriteOnce, on /mnt/data (or an EBS disk)" | "I need 500 Mi, ReadWriteOnce" |
| Created by | Cluster **admin** (static) or a **StorageClass** (dynamic) | **Developer**, next to their Deployment |
| Scope | **Cluster-wide** (no namespace) | **Namespaced** |

**Binding:** Kubernetes finds a PV that **satisfies** the PVC (enough size, the same access mode, the same StorageClass) and **binds** them one-to-one. The Pod then refers **only to the PVC**.

```
  Pod ──uses──▶ PVC (request: 500Mi, RWO) ──bound to──▶ PV (1Gi, RWO) ──▶ real disk (EBS / NFS / hostPath)
 (developer)        (developer)                         (admin / StorageClass)
```

> 💡 **Simple words mein:** PV = parking slot jo building (cluster) mein already bana hai. PVC = tumhari application: "mujhe ek slot chahiye, kam se kam itna bada". K8s match karke slot allot kar deta hai. Gaadi (Pod) sirf apna allotment number (PVC) jaanti hai.
>
> **Why two objects?** It **separates who provides storage from who uses it.** Developers don't need to know whether it's EBS, NFS or Azure Disk. They just ask for "500 Mi".

---

## 6. Static Provisioning: Write the PV Yourself

**The admin creates the PV first. The developer's PVC then binds to it.**

**PV (from class):**
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: student-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: ""            # ➕ added: "no class" (see the gotcha below)
  hostPath:
    path: "/mnt/data"             # fine for minikube; in the cloud this would be an EBS/NFS source
```

**PVC (from class):**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: student-pvc
spec:
  storageClassName: ""            # ➕ added: bind to a pre-made PV, don't create a new one
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
```

**Pod using the PVC:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: student-pod
spec:
  containers:
    - name: web
      image: nginx
      volumeMounts:
        - name: storage
          mountPath: /usr/share/nginx/html
  volumes:
    - name: storage
      persistentVolumeClaim:
        claimName: student-pvc
```

```bash
kubectl apply -f pv.yaml -f pvc.yaml -f pod.yaml
kubectl get pv,pvc
# NAME         CAPACITY  ACCESS MODES  RECLAIM POLICY  STATUS  CLAIM
# student-pv   1Gi       RWO           Retain          Bound   default/student-pvc
# NAME          STATUS  VOLUME      CAPACITY
# student-pvc   Bound   student-pv  1Gi          ← asked for 500Mi, got the WHOLE 1Gi PV

# Prove persistence
kubectl exec student-pod -- sh -c 'echo "<h1>persisted</h1>" > /usr/share/nginx/html/index.html'
kubectl delete pod student-pod && kubectl apply -f pod.yaml
kubectl exec student-pod -- cat /usr/share/nginx/html/index.html    # <h1>persisted</h1> ✅
```

> ⚠️ **Note 1: a PVC gets the whole PV.** Binding is one-to-one, so the 500 Mi claim takes all of the 1 Gi PV. The other 524 Mi can't be used by anyone else.
>
> ⚠️ **Note 2 (minikube / cloud gotcha):** if a cluster has a **default StorageClass** (minikube's `standard`, EKS's `gp2`/`gp3`), a PVC **without** `storageClassName` is given that default. It then **dynamically creates a new PV** instead of binding to your `student-pv`, so your PV sits `Available` and unused. Setting `storageClassName: ""` on **both** objects forces static binding.

---

## 7. Dynamic Provisioning: StorageClass

Writing PVs by hand doesn't scale. A **StorageClass** describes a **type of storage** plus a **provisioner** that **creates PVs automatically** whenever a PVC asks for one.

```
  PVC (storageClassName: standard, 500Mi) ──▶ StorageClass "standard" ──provisioner──▶ creates a 500Mi PV + real disk
                                                                                        and binds it to the PVC ✅
```

**PVC (from class): no PV needed at all**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
  storageClassName: standard        # leave it out to use the cluster's default class
```

**➕ What a StorageClass looks like** (usually already provided by the cluster):
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com            # who creates the disks (minikube: k8s.io/minikube-hostpath)
parameters:
  type: gp3                             # provisioner-specific settings
reclaimPolicy: Delete                   # dynamic PVs default to Delete
volumeBindingMode: WaitForFirstConsumer # create the disk only when a Pod uses it, in that Pod's zone
allowVolumeExpansion: true              # PVCs can be resized later
```

```bash
kubectl get storageclass          # (default) is marked next to one name
kubectl apply -f dynamic-pvc.yaml
kubectl get pvc,pv                # a PV named pvc-xxxx appears automatically, sized exactly 500Mi
```

| | Static | Dynamic |
|-|--------|---------|
| Who creates the PV | Admin, by hand, in advance | **StorageClass provisioner**, on demand |
| PV size | Whatever the admin made (PVC may get more) | **Exactly** what the PVC asked for |
| Effort / scale | High / poor | Low / great |
| Typical use | Labs, existing NFS shares | **Almost all real clusters** |

> 💡 **Simple words mein:** static = pehle se bane kamre, jo khaali mile wahi do. Dynamic = order pe kamra banta hai, bilkul utne size ka jitna maanga.

---

## 8. Access Modes, Reclaim Policies & PV Phases

### Access modes
| Mode | Short | Meaning | Typical storage |
|------|-------|---------|-----------------|
| **ReadWriteOnce** | RWO | Read-write, mounted by **one node** at a time | Block disks: AWS EBS, GCE PD, hostPath |
| **ReadOnlyMany** | ROX | Read-only, **many nodes** | Shared static content |
| **ReadWriteMany** | RWX | Read-write, **many nodes** | **File shares**: NFS, AWS EFS, Azure Files |
| ➕ ReadWriteOncePod | RWOP | Read-write by **one Pod** only | Strict single-writer apps |

> ⚠️ RWO means **one node**, not one Pod. Several Pods on the **same** node can share it. A Deployment with replicas spread across nodes needs **RWX** storage to share one volume.

### Reclaim policy: what happens to the PV when its PVC is **deleted**
| Policy | Effect | Default for |
|--------|--------|-------------|
| **Retain** | PV and data **kept**. PV becomes `Released` and needs **manual cleanup** before reuse. | Static PVs |
| **Delete** | PV **and the real disk** are deleted | Dynamically provisioned PVs |
| ~~Recycle~~ | (Deprecated) wiped the files and reused the PV | — |

> ⚠️ For important data (production DBs), use **Retain**, so deleting a PVC by mistake doesn't destroy the disk.

### PV phases (`kubectl get pv` → STATUS)
```
 Available ──(PVC binds)──▶ Bound ──(PVC deleted)──▶ Released ──(admin cleans / deletes)──▶ ...
                                                     (Retain: data still there, NOT reusable automatically)
 Failed = automatic reclamation failed
```
PVC phases: **Pending** (no matching PV yet, or waiting for its first Pod) → **Bound**.

> **Debug a `Pending` PVC:** `kubectl describe pvc <name>` → Events. Usual causes: no PV is big enough, the access mode doesn't match, the StorageClass is different, or (with `WaitForFirstConsumer`) no Pod uses it yet.

---

## 9. Comparing the Volume Types

| | `emptyDir` | `hostPath` | PV + PVC |
|-|------------|------------|----------|
| Data lives | Node disk (or RAM), per Pod | A fixed path on **one node** | **External storage** (EBS, NFS, …) |
| Survives container restart | ✅ | ✅ | ✅ |
| Survives Pod deletion | ❌ | ✅ (same node) | ✅ |
| Survives Pod moving to another node | ❌ | ❌ | ✅ (storage follows the Pod) |
| Shared between containers in a Pod | ✅ | ✅ | ✅ |
| Production app data | ❌ | ❌ ⚠️ | ✅ |
| Use for | Cache, scratch, sidecar sharing | Node agents, single-node labs | **Databases, uploads, anything that must persist** |

**Docker ↔ Kubernetes mapping** ([L7 §8](07-Docker-Networking-and-Volumes.md#8-named-volumes-vs-bind-mounts)):
| Docker | Kubernetes (closest match) |
|--------|----------------------------|
| Bind mount (`-v $(pwd)/dir:/path`) | `hostPath` |
| `tmpfs` | `emptyDir: { medium: Memory }` |
| Named volume (`-v mydata:/path`) | PVC (backed by a PV) |

> ➕ Databases in K8s usually run as a **StatefulSet** with `volumeClaimTemplates`, which give **each replica its own PVC** (`data-db-0`, `data-db-1`, …), paired with a **headless Service**.

---

## 10. Horizontal Pod Autoscaler (HPA)

**HPA automatically changes the number of Pod replicas** of a Deployment (or ReplicaSet / StatefulSet) based on **observed metrics**, usually CPU utilization.

```
 Load ↑  → CPU above target → HPA raises replicas (up to maxReplicas)
 Load ↓  → CPU below target → HPA lowers replicas (down to minReplicas, after a cool-down)
```

> 💡 **Simple words mein:** restaurant mein rush aaya → manager (HPA) aur waiters (Pods) bula leta hai. Rush khatam → kuch der wait karke extra waiters ghar bhej deta hai.

### How it works
```
 kubelet (cAdvisor) ──▶ metrics-server ──▶ Metrics API ◀── HPA controller (in the Controller Manager)
                                                              │ checks every 15 s
                                                              ▼
                                         desired = ceil( current × currentCPU% / targetCPU% )
                                                              │
                                                              ▼
                                            updates Deployment.spec.replicas (min ≤ n ≤ max)
```

**Worked example:** 2 replicas, average CPU **150%** of their request, target **50%** → `ceil(2 × 150 / 50)` = **6** replicas (capped at `maxReplicas`).

### Two requirements (the most common reasons HPA "doesn't work")
| Requirement | Why | Missing → |
|-------------|-----|-----------|
| **metrics-server** installed | HPA needs live CPU numbers | `TARGETS: <unknown>/50%` |
| **`resources.requests.cpu`** set on the containers | Utilization % = usage ÷ **request**. With no request, there's nothing to divide by. | `<unknown>`, and HPA does nothing |

```bash
minikube addons enable metrics-server
kubectl top pods          # works = metrics are flowing
```

### HPA YAML from class (`autoscaling/v1`, CPU only)
```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: my-hpa
spec:
  scaleTargetRef:                     # WHAT to scale
    apiVersion: apps/v1
    kind: Deployment
    name: my-deployment
  minReplicas: 1
  maxReplicas: 5
  targetCPUUtilizationPercentage: 50  # keep average CPU ≈ 50% of the requested CPU
```

### ➕ The modern version (`autoscaling/v2`): same behaviour, more metrics
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: my-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: my-deployment
  minReplicas: 1
  maxReplicas: 5
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 50
    - type: Resource                  # v2 can use several metrics; the HIGHEST result wins
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 70
  behavior:                           # optional tuning
    scaleDown:
      stabilizationWindowSeconds: 300 # wait 5 min of low load before removing Pods (the default)
```

**Quick way without YAML:**
```bash
kubectl autoscale deployment my-deployment --cpu-percent=50 --min=1 --max=5
kubectl get hpa
# NAME     REFERENCE                  TARGETS   MINPODS   MAXPODS   REPLICAS
# my-hpa   Deployment/my-deployment   12%/50%   1         5         1
```

> ⚠️ **Don't fight the HPA:** once an HPA manages a Deployment, **remove `replicas:` from the Deployment YAML** (or never re-apply it). Otherwise every `kubectl apply` resets the count, and the HPA has to scale it back up again.
>
> ➕ **Scale-up is fast, scale-down is slow (5 min by default).** This avoids "flapping" up and down on spiky traffic.

---

## 11. Hands-on Lab: Watch HPA Scale Under Load

```bash
# 0. Metrics
minikube addons enable metrics-server

# 1. A CPU-hungry app WITH a CPU request (required!) + a Service
kubectl create deployment php-apache --image=registry.k8s.io/hpa-example
kubectl set resources deployment php-apache --requests=cpu=200m --limits=cpu=500m
kubectl expose deployment php-apache --port=80

# 2. HPA: keep CPU at 50%, between 1 and 10 Pods
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10

# 3. Watch (in a second terminal)
kubectl get hpa php-apache --watch

# 4. Generate load (third terminal). Corrected command, see the note below.
kubectl run load-generator -i --tty --rm --image=busybox:1.36 --restart=Never -- \
  /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"

# Watch TARGETS climb (e.g. 250%/50%) and REPLICAS grow 1 → 4 → 6 ...
# 5. Ctrl+C the load generator → after ~5 min, replicas drop back to 1
kubectl get deploy php-apache
```

> ⚠️ **Fix to the class command:** `kubectl run --generator=run-pod/v1 ...` **no longer works**. The `--generator` flag was removed in newer kubectl versions, where `kubectl run` always creates a plain Pod. Use the command above: `-i --tty` to see output, `--rm` to delete the Pod when you exit, `--restart=Never` for a plain Pod.

---

## 12. ➕ HPA vs. VPA vs. Cluster Autoscaler

| Autoscaler | Changes | Example |
|------------|---------|---------|
| **HPA** (Horizontal Pod Autoscaler) | **Number of Pods** | 2 → 8 web Pods during a sale |
| **VPA** (Vertical Pod Autoscaler) | **CPU/memory requests of each Pod** (bigger Pods) | Raises a Pod's request from 200m to 600m CPU |
| **Cluster Autoscaler** / Karpenter | **Number of nodes** | HPA wants 8 Pods but nodes are full → adds a new VM |

> They work together: HPA adds Pods → some Pods stay `Pending` (no room) → the Cluster Autoscaler adds a node → the Pods get scheduled. Don't use HPA and VPA on the **same metric** (CPU) for the same app, because they work against each other.

---

## 13. Quick Revision Cheat Sheet

```
RECAP         ConfigMap = non-sensitive · Secret = sensitive (base64 ≠ encryption) · get / describe / delete
              Services: ClusterIP · NodePort · LoadBalancer · ExternalName · Headless (clusterIP: None)
VOLUME USE    spec.volumes (declare) + containers[].volumeMounts (mount at path)
EMPTYDIR      created with Pod, deleted with Pod · survives container restart · cache / scratch / share between containers
HOSTPATH      node's folder · survives Pod delete on SAME node · lost if Pod moves · security risk · labs / DaemonSets
PV            actual storage, cluster-wide, made by admin (static) or StorageClass (dynamic)
PVC           request (size + access mode), namespaced · Pod uses claimName · binds 1:1 to a PV
STATIC        admin writes PV · PVC binds · PVC may get MORE than asked · storageClassName: "" on both
DYNAMIC       PVC + storageClassName → provisioner creates exact-size PV · default class if omitted
ACCESS        RWO (one NODE) · ROX · RWX (many nodes, needs NFS/EFS) · RWOP (one Pod)
RECLAIM       Retain (keep data, PV → Released, manual) · Delete (default for dynamic) · Recycle deprecated
PHASES        PV: Available → Bound → Released (Failed) · PVC: Pending → Bound
HPA           scales replicas on metrics · desired = ceil(current × current% / target%)
              needs metrics-server + resources.requests.cpu (else <unknown>)
              v1: targetCPUUtilizationPercentage · v2: metrics[] (cpu, memory, …) + behavior
              kubectl autoscale deploy x --cpu-percent=50 --min=1 --max=5 · scale-down waits 5 min
LOAD TEST     kubectl run load -i --tty --rm --image=busybox --restart=Never -- sh -c "while ...wget..."
              (--generator flag removed)
OTHERS ➕     VPA = bigger Pods · Cluster Autoscaler = more nodes
```

---

## 14. Likely Exam / Interview Questions

1. **Why do Pods need volumes?** Container filesystems are reset on restart and lost when the Pod is removed. Volumes keep data longer and let containers share files.
2. **`emptyDir` vs. `hostPath`?** `emptyDir` lives and dies with the Pod and is used for scratch space or sharing between containers. `hostPath` mounts a node folder, which survives the Pod but is tied to that node and is a security risk.
3. **Does `emptyDir` data survive a container crash? A Pod deletion?** Yes for a crash, since the Pod is still alive. No for a deletion.
4. **Why is `hostPath` discouraged for application data?** The data stays on one node (a rescheduled Pod loses it), and it gives the Pod access to the node's filesystem.
5. **PV vs. PVC?** A PV is a piece of actual cluster storage, provided by an admin or a StorageClass. A PVC is a namespaced request for storage. Kubernetes binds them one-to-one.
6. **Why does Kubernetes separate PV and PVC?** It separates who provides storage from who uses it. Apps ask for size and access mode without knowing the storage backend.
7. **A PVC asks for 500 Mi and binds to a 1 Gi PV. What capacity does it get?** The whole 1 Gi. Binding is one-to-one, and the extra space can't be used by others.
8. **Static vs. dynamic provisioning?** Static means the admin creates PVs in advance. Dynamic means a StorageClass provisioner creates a correctly sized PV whenever a PVC asks for one.
9. **Your PVC created a new PV instead of binding to the one you made. Why?** The cluster has a default StorageClass, so the PVC was provisioned dynamically. Set `storageClassName: ""` (or a matching class) on both.
10. **Explain RWO, ROX and RWX.** Read-write by one node, read-only by many nodes, read-write by many nodes (needs file storage like NFS or EFS).
11. **Retain vs. Delete reclaim policy?** Retain keeps the PV and its data after the PVC is deleted (the PV becomes Released). Delete removes the PV and the underlying disk.
12. **What does an HPA do, and what does it need to work?** It scales a Deployment's replica count between min and max based on metrics. It needs metrics-server and CPU requests on the containers.
13. **An HPA shows `TARGETS <unknown>/50%`. What do you check?** That metrics-server is running (`kubectl top pods`), and that the containers have `resources.requests.cpu`.
14. **4 Pods run at an average of 90% CPU with a target of 60%. How many replicas will the HPA want?** ceil(4 × 90 / 60) = 6.
15. **`autoscaling/v1` vs. `autoscaling/v2`?** v1 supports CPU only (`targetCPUUtilizationPercentage`). v2 supports multiple metrics (CPU, memory, custom) and `behavior` tuning.
16. **Why does an HPA scale down slowly?** A stabilization window (5 min by default) prevents flapping on spiky traffic.
17. **HPA vs. VPA vs. Cluster Autoscaler?** More Pods vs. bigger Pods vs. more nodes.
