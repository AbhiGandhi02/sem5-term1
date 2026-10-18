# Kubernetes & Helm: Revision Class

> **DevOps – Lecture 14 notes (revision).** This class revised Lectures 9–13. So this file is a **compact revision sheet**: every heading links to the full notes. Only **StatefulSets** (§2) and the **end-to-end example** (§8) are new material. Anything marked **➕ Extra** was not covered in class.
>
> **Full notes:** [L9 Services & Deployments](09-Kubernetes-Objects-Services-Deployments.md) · [L10 Strategies](10-Kubernetes-Deployment-Strategies.md) · [L11 ConfigMaps, Secrets & Ingress](11-Kubernetes-ConfigMaps-Secrets-Ingress.md) · [L12 Volumes & HPA](12-Kubernetes-Volumes-and-Autoscaling.md) · [L13 Troubleshooting & Helm](13-Kubernetes-Troubleshooting-and-Helm.md)
>
> **Practice:** [Kubernetes MCQs](Kubernetes-MCQs.md) · **Next:** [Lecture 15 – Course Recap & CI/CD with GitHub Actions](15-CICD-with-GitHub-Actions.md).

---

## Table of Contents
1. [Workload Types at a Glance](#1-workload-types-at-a-glance)
2. [StatefulSets (new)](#2-statefulsets-new)
3. [ConfigMaps & Secrets](#3-configmaps--secrets)
4. [Ingress Routing](#4-ingress-routing)
5. [Service Types: Corrected Comparison](#5-service-types-corrected-comparison)
6. [Storage](#6-storage)
7. [Helm Commands](#7-helm-commands)
8. [End-to-End: nginx with YAML, then with Helm](#8-end-to-end-nginx-with-yaml-then-with-helm)
9. [Quick Revision Cheat Sheet](#9-quick-revision-cheat-sheet)
10. [Likely Exam / Interview Questions](#10-likely-exam--interview-questions)

---

## 1. Workload Types at a Glance

| Object | For | Pods are… | Example |
|--------|-----|-----------|---------|
| **[Deployment](09-Kubernetes-Objects-Services-Deployments.md#6-deployments)** | **Stateless** apps | Identical, interchangeable, random names (`web-7d9f-x2k`) | nginx, REST APIs, frontends |
| **[StatefulSet](#2-statefulsets-new)** | **Stateful** apps | Unique, **stable name + own storage** (`db-0`, `db-1`) | PostgreSQL, MongoDB, Kafka |
| **[DaemonSet](09-Kubernetes-Objects-Services-Deployments.md#11-daemonsets)** | Node agents | **One per node** | Fluentd, node-exporter |
| ➕ [Job](08-Kubernetes-Basics.md#12-hands-on-a-run-once-task-pod) / CronJob | Run-to-completion tasks | Run, finish, stop | DB migration, nightly backup |

> 💡 **Simple words mein:** Deployment ke Pods = cab drivers. Koi bhi aaye, kaam same. StatefulSet ke Pods = bank lockers. Locker no. 0 ka saaman hamesha locker no. 0 mein hi milega.

---

## 2. StatefulSets (new)

The class notes say a StatefulSet keeps "state consistent across restarts". More precisely, it gives every Pod **three things a Deployment doesn't**:

| Guarantee | Deployment | StatefulSet |
|-----------|------------|-------------|
| **Stable name** | Random (`web-7d9f-x2k`), new name when replaced | Ordinal: `db-0`, `db-1`, `db-2`. A replaced `db-1` is **still `db-1`** |
| **Stable network identity** | Reached only through the Service | Each Pod gets its own DNS name via a **headless Service**: `db-0.db-headless` |
| **Own persistent storage** | All replicas share one PVC (or none) | **One PVC per Pod** (`data-db-0`, `data-db-1`), via `volumeClaimTemplates`. It reattaches to the same Pod after a restart. |
| **Order** | All Pods start/stop together | Created **0 → 1 → 2**, deleted **2 → 1 → 0** |

```yaml
apiVersion: v1
kind: Service
metadata:
  name: db-headless
spec:
  clusterIP: None              # headless → per-Pod DNS records
  selector: { app: db }
  ports:
    - port: 5432
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: db
spec:
  serviceName: db-headless     # links Pods to the headless Service for DNS
  replicas: 3
  selector:
    matchLabels: { app: db }
  template:
    metadata:
      labels: { app: db }
    spec:
      containers:
        - name: postgres
          image: postgres:16
          envFrom:
            - secretRef: { name: db-secret }
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:        # one PVC per Pod, created automatically
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
```
```bash
kubectl get pods -l app=db      # db-0, db-1, db-2 (created in order)
kubectl get pvc                 # data-db-0, data-db-1, data-db-2
kubectl delete pod db-1         # comes back as db-1, with data-db-1 reattached ✅
```
> ⚠️ Deleting or scaling down a StatefulSet **does not delete its PVCs**, so the data stays safe. Clean them up yourself.
>
> ➕ In practice, teams often install databases with a **Helm chart** (e.g. a PostgreSQL chart), which builds this StatefulSet for you.

---

## 3. ConfigMaps & Secrets
Full notes: [L11 §3–7](11-Kubernetes-ConfigMaps-Secrets-Ingress.md#3-configmaps)

**The examples from class:**
```bash
kubectl create configmap app-config --from-literal=ENV=production
kubectl create secret generic app-secret \
  --from-literal=DB_PASSWORD='S3cr3t!' \
  --from-literal=JWT_SECRET='a-long-random-string'
```
```yaml
# used in a container
envFrom:
  - configMapRef: { name: app-config }    # ENV=production
  - secretRef:    { name: app-secret }    # DB_PASSWORD, JWT_SECRET (decoded automatically)
```
| ConfigMap | Secret |
|-----------|--------|
| Non-sensitive: `ENV`, log level, URLs | Sensitive: DB passwords, JWT/OAuth tokens, keys |
| Plain text | **Base64, which is not encryption.** Protected by RBAC + encryption at rest |

---

## 4. Ingress Routing
Full notes: [L11 §8–11](11-Kubernetes-ConfigMaps-Secrets-Ingress.md#8-ingress-why-we-need-it)

| | Path-based | Host-based |
|-|------------|------------|
| Rule | **Same domain**, different paths | **Different domains** |
| Example | `shop.com/api` → api-svc, `shop.com/` → web-svc | `api.shop.com` → api-svc, `shop.com` → web-svc |

**Why Ingress:** **one entry point (one load balancer)** for many Services, which saves cost, with routing rules and TLS managed in one place. It needs an **Ingress Controller** (e.g. NGINX) installed.

---

## 5. Service Types: Corrected Comparison
Full notes: [L9 §9](09-Kubernetes-Objects-Services-Deployments.md#9-service-types)

| Service type | Description | Reachable from outside the cluster? |
|--------------|-------------|--------------------------------------|
| **ClusterIP** (default) | Internal virtual IP for Pod-to-Pod traffic | ❌ No |
| **NodePort** | Static port (30000–32767) on every node's IP | ✅ Yes, `<nodeIP>:<nodePort>` |
| **LoadBalancer** | Cloud provider's load balancer in front of NodePorts | ✅ Yes, via the cloud LB's IP |
| **ExternalName** | DNS alias (CNAME) from inside the cluster **to** an outside name | ❌ **No**. It lets Pods reach something external, it doesn't expose anything. |
| **Headless** | `clusterIP: None`: no virtual IP, DNS returns Pod IPs | ❌ No, direct Pod-to-Pod (used by StatefulSets) |

> ⚠️ **Two corrections to the class table:**
> 1. **ExternalName** isn't "externally accessible". Traffic goes **out** from your Pods to an external name (e.g. `db` → `mydb.rds.amazonaws.com`). Nobody outside can reach your app through it.
> 2. **Headless** isn't "type set to None". The **type** stays `ClusterIP`, and you set **`clusterIP: None`**.

---

## 6. Storage
Full notes: [L12](12-Kubernetes-Volumes-and-Autoscaling.md#9-comparing-the-volume-types)

| Type | Data survives… | Use |
|------|----------------|-----|
| **emptyDir** | Container restarts, but **not Pod deletion** | Cache, scratch, sharing between containers |
| **hostPath** | Pod deletion **on the same node only** | Single-node labs, node agents |
| **PV + PVC** | Everything: independent of Pods and nodes | Databases, uploads |

```bash
kubectl get pv                 # cluster-wide storage pieces: STATUS Available / Bound / Released
kubectl get pvc                # namespaced requests: Pending / Bound
kubectl describe pvc <name>    # why is it Pending?
```
**Flow:** Pod → `persistentVolumeClaim.claimName` → PVC → bound to a PV (static, or created by a **StorageClass** = dynamic).

---

## 7. Helm Commands
Full notes: [L13 §10–16](13-Kubernetes-Troubleshooting-and-Helm.md#10-helm-why-a-package-manager)

| Command | Does |
|---------|------|
| `helm create <chart>` | Scaffold a chart: `Chart.yaml`, `values.yaml`, `templates/` |
| `helm template <release> <chart-dir>` | **Render** the YAML locally, without installing (check it before deploying) |
| `helm install <release> <chart>` | Install → creates release **revision 1** |
| `helm list` | Releases in this namespace (`-A` for all) |
| `helm status <release>` | Release state + NOTES |
| `helm upgrade <release> <chart> --set k=v` | Apply changes → new revision |
| `helm history <release>` ➕ | All revisions |
| `helm rollback <release> <rev>` | Go back → creates a **new** revision with the old config |
| `helm uninstall <release>` | Remove everything the release created |

```bash
helm install  my-release my-chart/
helm upgrade  my-release my-chart/ --set replicaCount=3     # revision 2
helm rollback my-release 1                                  # revision 3 (= revision 1's config)
helm uninstall my-release
```

> ⚠️ **➕ Upgrade gotcha:** `helm upgrade` **without** your earlier `--set` flags goes back to the chart defaults. A later `helm upgrade my-release my-chart/` would quietly drop `replicaCount=3`. Keep overrides in a **values file** (`-f values-prod.yaml`) and pass it every time, or use `--reuse-values`.

**Multi-environment:** one chart + `values-dev.yaml` / `values-stage.yaml` / `values-prod.yaml`:
```bash
helm upgrade --install shop ./my-chart -f values-prod.yaml -n prod
```

---

## 8. End-to-End: nginx with YAML, then with Helm

### A. Plain YAML (`app.yaml`): everything from the revision in one file
```yaml
apiVersion: v1
kind: ConfigMap
metadata: { name: web-config }
data:
  ENV: production
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: web }
spec:
  replicas: 2
  selector: { matchLabels: { app: web } }
  template:
    metadata: { labels: { app: web } }
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports: [{ containerPort: 80 }]
          envFrom: [{ configMapRef: { name: web-config } }]
---
apiVersion: v1
kind: Service
metadata: { name: web-svc }
spec:
  selector: { app: web }
  ports: [{ port: 80, targetPort: 80 }]          # ClusterIP: Ingress is the only way in
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata: { name: web-ingress }
spec:
  ingressClassName: nginx
  rules:
    - host: web.local
      http:
        paths:
          - path: /
            pathType: Prefix
            backend: { service: { name: web-svc, port: { number: 80 } } }
```
```bash
kubectl apply -f app.yaml
kubectl get deploy,svc,ingress,cm
kubectl exec deploy/web -- env | grep ENV       # ENV=production
kubectl delete -f app.yaml                      # you must remember every file/object yourself
```

### B. The same app with Helm
```bash
helm create web-chart              # nginx Deployment + Service + (optional) Ingress already templated
```
`values-prod.yaml`:
```yaml
replicaCount: 2
image:
  tag: "1.27"
ingress:
  enabled: true
  className: nginx
  hosts:
    - host: web.local
      paths:
        - path: /
          pathType: Prefix
```
```bash
helm template web ./web-chart -f values-prod.yaml | less       # check the YAML first
helm install  web ./web-chart -f values-prod.yaml
helm upgrade  web ./web-chart -f values-prod.yaml --set replicaCount=4
helm rollback web 1
helm uninstall web                                             # one command removes it all ✅
```

| | Plain YAML | Helm |
|-|------------|------|
| Per-environment differences | Copy and edit files | One chart + a values file per environment |
| Install / remove | `apply` / `delete` each file | `helm install` / `helm uninstall` |
| History & rollback | Per Deployment only (`rollout undo`) | **Whole app** (`helm rollback`) |
| Good for | Learning, small apps | Real multi-object apps, CI/CD |

---

## 9. Quick Revision Cheat Sheet

```
WORKLOADS     Deployment = stateless, interchangeable · StatefulSet = stateful, db-0/db-1, own PVC, ordered
              DaemonSet = 1 per node · Job/CronJob = run to completion
STATEFULSET   serviceName → headless Service (per-Pod DNS) · volumeClaimTemplates → 1 PVC per Pod
              PVCs NOT deleted with the StatefulSet
CONFIG        ConfigMap ENV=production · Secret DB_PASSWORD / JWT (base64 ≠ encryption) · envFrom
INGRESS       path-based (same host, /api /web) · host-based (api.x.com, x.com) · 1 LB · needs controller
SERVICES      ClusterIP internal · NodePort nodeIP:30000-32767 · LoadBalancer cloud LB
              ExternalName = outbound DNS alias (NOT external access) · Headless = clusterIP: None
STORAGE       emptyDir (Pod life) · hostPath (node) · PV/PVC (independent) · get pv / get pvc
HELM          create · template · install · list · status · upgrade --set / -f · history · rollback · uninstall
              upgrade without your --set resets to defaults → use -f values file / --reuse-values
              values-dev / stage / prod.yaml + helm upgrade --install
```

---

## 10. Likely Exam / Interview Questions

1. **Deployment vs. StatefulSet?** A Deployment runs interchangeable Pods with random names and shared or no storage. A StatefulSet gives each Pod a stable name, its own DNS name and its own PVC, and starts and stops Pods in order.
2. **Why does a StatefulSet need a headless Service?** To give each Pod its own DNS record (`db-0.db-headless`), so clients can reach a specific replica.
3. **What does `volumeClaimTemplates` do?** It creates one PVC per StatefulSet Pod, which reattaches to the same Pod after a restart.
4. **When would you use a DaemonSet instead of a Deployment?** When exactly one Pod must run on every node, e.g. log or monitoring agents.
5. **Give an example of data for a ConfigMap and for a Secret.** `ENV=production` goes in a ConfigMap. A DB password or JWT secret goes in a Secret.
6. **Path-based vs. host-based routing?** The same domain with different paths vs. different domains.
7. **Which Service types expose an app outside the cluster?** NodePort and LoadBalancer (and Ingress, which sits in front of Services). ClusterIP, Headless and ExternalName don't.
8. **What does an ExternalName Service do?** It gives Pods an internal DNS alias for an external hostname. It doesn't expose anything.
9. **How is a headless Service defined?** `clusterIP: None`. The type is still ClusterIP.
10. **emptyDir vs. hostPath vs. PV?** Pod lifetime vs. node lifetime vs. independent, durable storage.
11. **What does `helm template` do, and why use it?** It renders the chart's YAML locally without installing, so you can review or debug it.
12. **You ran `helm upgrade --set replicaCount=3`, then a plain `helm upgrade`. How many replicas now?** The chart default, because `--set` values aren't remembered. Use a values file or `--reuse-values`.
13. **`helm rollback my-release 1` when the release is at revision 2. What revision number does it create?** Revision 3, holding revision 1's configuration.
14. **How does Helm handle dev, stage and prod?** One chart, with environment-specific values files passed through `-f`.
