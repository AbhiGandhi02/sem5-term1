# Kubernetes Part 4: ConfigMaps, Secrets & Ingress

> **DevOps – Lecture 11 notes.** Anything marked **➕ Extra** was not covered in class. It's there because it fills a gap or comes up often in labs and interviews.
>
> **Builds on:** [Lecture 8 – Kubernetes Basics](08-Kubernetes-Basics.md) (architecture) and [Lecture 9 – Services](09-Kubernetes-Objects-Services-Deployments.md#8-services-why-we-need-them) (ClusterIP, LoadBalancer, ports). DNS and HTTPS ideas come from [Lecture 3](03-Networking.md).
>
> **Practice:** [Kubernetes MCQs](Kubernetes-MCQs.md) · **Next:** [Lecture 12 – Volumes & Autoscaling](12-Kubernetes-Volumes-and-Autoscaling.md).

---

## Table of Contents
1. [Architecture Recap](#1-architecture-recap)
2. [Why Separate Config from Code?](#2-why-separate-config-from-code)
3. [ConfigMaps](#3-configmaps)
4. [Using a ConfigMap in a Pod](#4-using-a-configmap-in-a-pod)
5. [Secrets](#5-secrets)
6. [Using a Secret in a Pod](#6-using-a-secret-in-a-pod)
7. [ConfigMap vs. Secret](#7-configmap-vs-secret)
8. [Ingress: Why We Need It](#8-ingress-why-we-need-it)
9. [Ingress vs. Ingress Controller](#9-ingress-vs-ingress-controller)
10. [Host-Based Routing](#10-host-based-routing)
11. [Path-Based Routing](#11-path-based-routing)
12. [➕ HTTPS with Ingress (TLS)](#12--https-with-ingress-tls)
13. [Hands-on Lab: Config + Secret + Ingress on minikube](#13-hands-on-lab-config--secret--ingress-on-minikube)
14. [Quick Revision Cheat Sheet](#14-quick-revision-cheat-sheet)
15. [Likely Exam / Interview Questions](#15-likely-exam--interview-questions)

---

## 1. Architecture Recap

| Where | Component | Job | More in |
|-------|-----------|-----|---------|
| Master / control plane | **etcd** | Key-value DB holding **all** cluster data (ConfigMaps and Secrets live here too) | [L8 §4](08-Kubernetes-Basics.md#4-control-plane-master-node-components) |
| | **API Server** | Front end for every REST call, and the **single point of contact** for all components | |
| | **Controller Manager** | Runs controllers: ReplicaSet, Deployment, Node, … | |
| | **Scheduler** | Picks a node for each new Pod, based on resources and policies | |
| Worker node | **kubelet** | Runs and watches the node's containers, reacts to changes | [L8 §5](08-Kubernetes-Basics.md#5-worker-node-components) |
| | **kube-proxy** | Network rules that make **Services** work | |
| | **Container runtime (containerd)** | Actually runs the containers, through the **CRI** (Container Runtime Interface) | |

> ➕ **CRI** is the plug-in standard between the kubelet and a runtime. containerd and CRI-O both implement it, which is why K8s isn't tied to any one runtime.

---

## 2. Why Separate Config from Code?

The **same image** should run in dev, staging and prod. What changes between them is **configuration**: DB host, log level, feature flags, passwords.

| ❌ Bad: baked into the image | ✅ Good: injected at run time |
|------------------------------|------------------------------|
| `ENV DB_HOST=prod-db` in the Dockerfile | `DB_HOST` comes from a ConfigMap |
| A new image for every environment / every config change | **One image**, a different ConfigMap per environment |
| Passwords end up in image layers and Git | Passwords come from a Secret |

> 💡 **Simple words mein:** image = mobile phone, config = SIM card. Phone same rehta hai, SIM badal ke alag network (dev / prod) pe chala lo.

| Data type | Kubernetes object |
|-----------|-------------------|
| **Non-sensitive** (mode, log level, URLs, config files) | **ConfigMap** |
| **Sensitive** (passwords, API keys, tokens, TLS certs) | **Secret** |

---

## 3. ConfigMaps

A **ConfigMap** stores **non-sensitive key-value config** separately from your app code and image.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: example-config
data:
  APP_MODE: development
  LOG_LEVEL: debug
  app.properties: |              # ➕ a whole FILE as one value ("|" keeps the line breaks)
    cache.size=100
    feature.newUI=true
```

**Creating one from the command line** ➕
```bash
kubectl create configmap example-config \
  --from-literal=APP_MODE=development \
  --from-literal=LOG_LEVEL=debug
kubectl create configmap nginx-conf --from-file=nginx.conf   # key = file name, value = file content
kubectl create configmap app-env --from-env-file=.env.dev    # every KEY=value line becomes a key
```

```bash
kubectl get configmaps                  # or: kubectl get cm
kubectl describe cm example-config      # shows the data in plain text
kubectl get cm example-config -o yaml
```

> ➕ Limit: **1 MiB** per ConfigMap. It's for config, not data files.

---

## 4. Using a ConfigMap in a Pod

There are three ways to use one: **pick single keys**, **load all keys**, or **mount as files**.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-demo
spec:
  containers:
    - name: app
      image: busybox
      command: ["sh", "-c", "env | grep -E 'APP_|LOG_'; cat /etc/config/app.properties; sleep 3600"]

      # 1. One key → one env var (you can rename it)
      env:
        - name: APP_ENVIRONMENT          # name inside the container
          valueFrom:
            configMapKeyRef:
              name: example-config
              key: APP_MODE

      # 2. ALL keys → env vars with the same names
      envFrom:
        - configMapRef:
            name: example-config

      # 3. Mount as files: each key becomes a file
      volumeMounts:
        - name: config-vol
          mountPath: /etc/config
  volumes:
    - name: config-vol
      configMap:
        name: example-config
```
```bash
kubectl exec config-demo -- ls /etc/config
# APP_MODE  LOG_LEVEL  app.properties      ← one file per key
```

| Method | Best for | If the ConfigMap changes later… |
|--------|----------|----------------------------------|
| `env` / `envFrom` | Simple settings read at startup | ❌ **Not updated**. Env vars are fixed when the container starts → `kubectl rollout restart deploy/<name>` |
| **Volume** (files) | Whole config files (`nginx.conf`, `application.yml`) | ✅ Files update **automatically** after a short delay (~1 min). The app must re-read them. |

> ⚠️ A Pod that references a ConfigMap that **doesn't exist** won't start (`CreateContainerConfigError`). Create the ConfigMap first.

---

## 5. Secrets

A **Secret** stores **sensitive data**: passwords, OAuth tokens, API keys, TLS certificates.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: example-secret
type: Opaque                    # generic key-value secret (the default type)
data:
  password: cGFzc3dvcmQ=        # base64 of "password"
```

### ⚠️ Base64 is encoding, NOT encryption
The class notes say base64 "obscures" the data. Be precise: **anyone can decode it in one command.**
```bash
echo -n 'password' | base64          # cGFzc3dvcmQ=      (-n: no trailing newline!)
echo 'cGFzc3dvcmQ=' | base64 -d      # password
kubectl get secret example-secret -o jsonpath='{.data.password}' | base64 -d
```
Base64 exists only so that **binary data** (certificates, keys) fits in YAML. It is **not** security.

> 💡 **Simple words mein:** base64 = message ko ulta likh dena. Dekhne mein ajeeb, par koi bhi seedha karke padh lega. Taala (encryption) nahi hai.

### So what actually makes Secrets "secret"? ➕
| Protection | What it does |
|------------|--------------|
| **RBAC** | Only allowed users/apps can `get` Secrets (you can give developers ConfigMap access without Secret access) |
| **Encryption at rest** | The cluster admin can make the API server **encrypt Secrets inside etcd** (off by default in plain clusters, usually on in managed EKS/GKE/AKS) |
| **Kept in memory** | Mounted Secret files live in `tmpfs` (RAM) on the node, never written to the node's disk |
| **Not shown by default** | `kubectl describe secret` shows sizes only, not values |

### Creating Secrets
```bash
# From literals: kubectl does the base64 for you ✅
kubectl create secret generic db-secret \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASSWORD='S3cr3t!'

kubectl create secret generic tls-files --from-file=./server.key   # from a file
```
**➕ `stringData`:** write plain text in YAML, and K8s base64-encodes it on save:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  DB_USER: admin
  DB_PASSWORD: S3cr3t!
```

| ➕ Secret `type` | Used for |
|-----------------|----------|
| `Opaque` | Anything (default) |
| `kubernetes.io/tls` | TLS cert + key for HTTPS (used by Ingress, §12) |
| `kubernetes.io/dockerconfigjson` | Credentials to pull from a **private registry** (`imagePullSecrets`) |

> ⚠️ **Never commit Secret YAML to Git.** Base64 in a public repo = a leaked password. ➕ Teams use **Sealed Secrets**, **External Secrets Operator** (pulls from AWS Secrets Manager / Vault) or **SOPS** instead.

---

## 6. Using a Secret in a Pod

It works exactly like a ConfigMap, with `secretKeyRef` / `secretRef` / `secret:` instead.

```yaml
spec:
  containers:
    - name: app
      image: postgres:16
      env:
        - name: POSTGRES_PASSWORD          # one key
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: DB_PASSWORD
      envFrom:
        - secretRef:                       # all keys
            name: db-secret
      volumeMounts:
        - name: creds
          mountPath: /etc/creds
          readOnly: true
  volumes:
    - name: creds
      secret:
        secretName: db-secret              # note: secretName, not name
```
Inside the container, values arrive **already decoded** (plain text). The app never deals with base64.

> ➕ **Prefer volume mounts over env vars for secrets:** env vars can leak into crash dumps, debug logs and child processes, and they don't update without a restart.

---

## 7. ConfigMap vs. Secret

| | ConfigMap | Secret |
|-|-----------|--------|
| For | Non-sensitive config | Passwords, tokens, keys, certs |
| Stored in YAML as | Plain text (`data:`) | **Base64** (`data:`) or plain (`stringData:`) |
| Real protection | None needed | RBAC, optional encryption at rest, tmpfs mounts |
| `kubectl describe` shows values? | ✅ Yes | ❌ No (only sizes) |
| Used via | `configMapKeyRef`, `configMapRef`, `configMap` volume | `secretKeyRef`, `secretRef`, `secret` volume |
| Size limit | 1 MiB | 1 MiB |
| Create | `kubectl create configmap` | `kubectl create secret generic` |

---

## 8. Ingress: Why We Need It

Say you have 5 web apps. Exposing each with a `LoadBalancer` Service ([L9 §9](09-Kubernetes-Objects-Services-Deployments.md#9-service-types)) means:
- **5 cloud load balancers** → **5 bills** (each LB costs money every hour) and **5 public IPs** to manage.
- **No HTTP awareness**: a Service works at **Layer 4** (IP + port). It can't route by domain name or URL path.
- TLS certificates handled separately for each app.

**Ingress** = **one entry point** (one load balancer) with **HTTP rules** that send each request to the right Service, based on **hostname** or **URL path**.

```
 WITHOUT Ingress                             WITH Ingress
 app1.com ──▶ LB #1 ──▶ service1             app1.com ─┐                 ┌─▶ service1
 app2.com ──▶ LB #2 ──▶ service2             app2.com ─┼─▶ ONE LB ──▶ Ingress ─┼─▶ service2
 shop.com ──▶ LB #3 ──▶ service3             shop.com/api ┘   Controller     └─▶ service3
   3 LBs, 3 bills                              1 LB, rules decide, 1 bill
```

> 💡 **Simple words mein:** Ingress = mall ka ek main gate + receptionist. Har dukaan ka alag gate banane ki jagah, ek gate pe receptionist bataata hai: "app1 chahiye? left jao. /api? upar jao."

| | NodePort | LoadBalancer | **Ingress** |
|-|----------|--------------|-------------|
| OSI layer | L4 | L4 | **L7 (HTTP/HTTPS)** |
| Route by host / path | ❌ | ❌ | ✅ |
| Public IPs / LBs | Node IPs, odd ports | **One per Service** | **One for many Services** |
| TLS termination | ❌ | Depends on cloud | ✅ Central |
| Protocols | Any TCP/UDP | Any TCP/UDP | **HTTP/HTTPS only** |

---

## 9. Ingress vs. Ingress Controller

| | **Ingress** (resource) | **Ingress Controller** |
|-|------------------------|------------------------|
| What | A **YAML object**: the routing **rules** | A **running program** (Pods + a Service) that **reads** those rules and **routes traffic** |
| Analogy | The **rule book** | The **traffic police** enforcing it |
| Comes with K8s? | ✅ The API type exists | ❌ **You must install one** |
| Examples | `kind: Ingress` | **NGINX Ingress Controller** (most common), Traefik, HAProxy, AWS Load Balancer Controller, GKE Ingress |

> ⚠️ **Classic trap:** you apply an Ingress, but **no controller is installed** → it's accepted, but **nothing happens**: no address, no routing. The rules have no one to enforce them.

```
 Internet ──▶ cloud LB (one) ──▶ Ingress Controller Pods (nginx) ──reads rules──▶ Ingress objects
                                         │
                                         ├──▶ service1 (ClusterIP) ──▶ Pods
                                         └──▶ service2 (ClusterIP) ──▶ Pods
```
Note that the backend Services can stay plain **ClusterIP**. Only the controller is exposed.

**Installing one:**
```bash
minikube addons enable ingress                       # minikube
kubectl get pods -n ingress-nginx                    # controller Pods running?
kubectl get ingressclass                             # ➕ e.g. "nginx"
```
➕ If a cluster has several controllers, pick one per Ingress with `spec.ingressClassName: nginx`.

---

## 10. Host-Based Routing

**Different domain names → different Services**, all on **one** IP.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example-ingress
spec:
  ingressClassName: nginx             # ➕ which controller (optional if there's a default)
  rules:
    - host: app1.example.com          # requests with "Host: app1.example.com"
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: service1
                port:
                  number: 80          # the SERVICE's port, not the container's
    - host: app2.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: service2
                port:
                  number: 80
```

**How does one IP serve two domains?** Both DNS names point to the **same** IP. The browser sends the domain in the HTTP **`Host` header**, and the controller reads that header to choose the backend.
```bash
curl -H "Host: app1.example.com" http://<ingress-ip>/    # → service1
curl -H "Host: app2.example.com" http://<ingress-ip>/    # → service2
```

---

## 11. Path-Based Routing

**One domain, different URL paths → different Services.**

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-ingress
spec:
  ingressClassName: nginx
  rules:
    - host: myapp.local
      http:
        paths:
          - path: /frontend
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
          - path: /backend
            pathType: Prefix
            backend:
              service:
                name: backend-service
                port:
                  number: 80
```
```
 myapp.local/frontend        → frontend-service
 myapp.local/frontend/about  → frontend-service   (Prefix matches sub-paths)
 myapp.local/backend/users   → backend-service
 myapp.local/other           → 404 from the controller's default backend
```

### `pathType`
| Value | Matches | `/backend` rule matches `/backend/users`? |
|-------|---------|-------------------------------------------|
| **`Prefix`** | The path and everything under it, split on `/` | ✅ Yes (but **not** `/backendX`) |
| **`Exact`** | Only that exact path | ❌ No |
| `ImplementationSpecific` ➕ | Up to the controller | — |

If several rules match, the **longest matching path wins**.

> ⚠️ **➕ Gotcha: the path is passed through as it is.** A request for `/backend/users` reaches `backend-service` as **`/backend/users`**, not `/users`. If the app only knows `/users`, it returns 404. Either build the app to serve under `/backend`, or ask NGINX to strip the prefix:
> ```yaml
> metadata:
>   annotations:
>     nginx.ingress.kubernetes.io/rewrite-target: /$2
> spec:
>   rules:
>     - http:
>         paths:
>           - path: /backend(/|$)(.*)
>             pathType: ImplementationSpecific
> ```

**Host vs. path routing:**
| Host-based | Path-based |
|------------|------------|
| `app1.example.com`, `app2.example.com` | `example.com/frontend`, `example.com/backend` |
| Separate products / teams / sub-domains | Parts of one app (UI + API) on one domain |

You can **combine** them: several hosts, each with several paths.

---

## 12. ➕ HTTPS with Ingress (TLS)

The Ingress controller can **terminate TLS** for every app in one place. The backends can then stay plain HTTP inside the cluster.

```bash
kubectl create secret tls app1-tls --cert=app1.crt --key=app1.key   # a Secret of type kubernetes.io/tls
```
```yaml
spec:
  tls:
    - hosts:
        - app1.example.com
      secretName: app1-tls           # cert + key from the Secret
  rules:
    - host: app1.example.com
      ...
```
> In production, **cert-manager** + **Let's Encrypt** create and renew these Secrets automatically. It's the Kubernetes version of Certbot ([L3 §10](03-Networking.md#10-http-https--tls)).

---

## 13. Hands-on Lab: Config + Secret + Ingress on minikube

```bash
# 0. Enable the NGINX Ingress Controller
minikube addons enable ingress

# 1. Config + Secret
kubectl create configmap web-config --from-literal=APP_MODE=development --from-literal=LOG_LEVEL=debug
kubectl create secret generic web-secret --from-literal=API_KEY=abc123

# 2. Two simple apps + ClusterIP Services
kubectl create deployment frontend --image=hashicorp/http-echo -- /http-echo -text="hello from FRONTEND" -listen=:5678
kubectl create deployment backend  --image=hashicorp/http-echo -- /http-echo -text="hello from BACKEND"  -listen=:5678
kubectl expose deployment frontend --name=frontend-service --port=80 --target-port=5678
kubectl expose deployment backend  --name=backend-service  --port=80 --target-port=5678

# 3. Inject the config into backend and check it
kubectl set env deployment/backend --from=configmap/web-config
kubectl set env deployment/backend --from=secret/web-secret
kubectl set env deployment/backend --list   # lists the injected vars (http-echo has no shell, so no exec)

# 4. Apply the path-based Ingress from §11 (saved as path-ingress.yaml)
kubectl apply -f path-ingress.yaml
kubectl get ingress                     # wait until ADDRESS is filled in

# 5. Point myapp.local at the Ingress, then test
echo "$(minikube ip) myapp.local" | sudo tee -a /etc/hosts
curl http://myapp.local/frontend        # hello from FRONTEND
curl http://myapp.local/backend         # hello from BACKEND
curl http://myapp.local/other           # 404 Not Found (default backend)
```
> On **Docker Desktop for Mac**, `minikube ip` isn't reachable. Run `minikube tunnel` in another terminal and use `127.0.0.1 myapp.local` in `/etc/hosts`.
>
> **Debugging:** `kubectl describe ingress path-ingress` (rules + backends + events), and `kubectl logs -n ingress-nginx deploy/ingress-nginx-controller` (every request the controller handled).

---

## 14. Quick Revision Cheat Sheet

```
ARCH          etcd (all data, incl. ConfigMaps/Secrets) · API server (single contact) · controller mgr · scheduler
              kubelet · kube-proxy · containerd via CRI
WHY CONFIG    one image, many environments → inject config at run time
CONFIGMAP     non-sensitive key/values or whole files · kubectl create cm x --from-literal / --from-file
USE IT        env.valueFrom.configMapKeyRef (one key) · envFrom.configMapRef (all) · volume (files)
              env = fixed at start (rollout restart) · volume files auto-update (~1 min)
SECRET        sensitive data · type Opaque / kubernetes.io/tls / dockerconfigjson
              data: base64 (ENCODING, NOT ENCRYPTION) · stringData: plain text · echo -n x | base64
              real protection = RBAC + encryption at rest + tmpfs · never commit to Git
USE IT        secretKeyRef · secretRef · volume (secretName) · arrives decoded in the container
INGRESS       L7 HTTP routing · ONE LB for many Services → saves cost · backends stay ClusterIP
CONTROLLER    actually routes (NGINX most common) · must be installed · no controller = rules do nothing
HOST-BASED    rules[].host → different domains, same IP · uses the HTTP Host header
PATH-BASED    paths[].path → /frontend, /backend on one domain · Prefix (sub-paths) vs Exact · longest wins
              path is forwarded unchanged → rewrite-target annotation to strip it ➕
TLS ➕        spec.tls + Secret type kubernetes.io/tls · cert-manager + Let's Encrypt
```

---

## 15. Likely Exam / Interview Questions

1. **Why keep configuration out of the container image?** So one image can run in every environment, and config changes don't need a rebuild. It also keeps secrets out of image layers.
2. **ConfigMap vs. Secret?** ConfigMaps hold non-sensitive config in plain text. Secrets hold sensitive data, encoded in base64 and protected by RBAC and optional encryption at rest.
3. **Is base64 encryption? So how is a Secret protected?** No, it's reversible encoding. Real protection comes from RBAC, encryption at rest in etcd, tmpfs mounts, and keeping Secrets out of Git.
4. **What are three ways to use a ConfigMap in a Pod?** One key as an env var (`configMapKeyRef`), all keys as env vars (`envFrom`), or mounted as files (a volume).
5. **You updated a ConfigMap, but the app still sees old values. Why?** It reads them as env vars, which are fixed at container start. Restart (`kubectl rollout restart`) or mount the ConfigMap as a volume.
6. **`data` vs. `stringData` in a Secret?** `data` needs base64 values. `stringData` takes plain text, and Kubernetes encodes it.
7. **Why might you mount a Secret as a file rather than an env var?** Env vars can leak into logs and child processes and don't update. Files are kept in tmpfs and can be refreshed.
8. **What is an Ingress, and what problem does it solve?** L7 HTTP routing rules that let many Services share one load balancer, routing by host or path. That saves cost and centralizes TLS.
9. **Ingress vs. Ingress Controller?** The Ingress is the rules (a YAML object). The controller (e.g. NGINX) is the running software that enforces them, and it must be installed.
10. **You created an Ingress, but it has no ADDRESS and traffic doesn't route. Most likely cause?** No Ingress controller is installed, or the `ingressClassName` is wrong.
11. **Host-based vs. path-based routing?** Host-based routes different domain names to different Services. Path-based routes different URL paths on one domain.
12. **How does host-based routing work with a single IP?** Each domain's DNS points to the same IP, and the controller routes by the HTTP `Host` header.
13. **`pathType: Prefix` vs. `Exact`?** Prefix matches the path and its sub-paths. Exact matches only that path.
14. **A request to `/backend/users` returns 404 from the backend app, even though the Ingress rule matches. Why?** The path is forwarded unchanged and the app doesn't serve `/backend/...`. Fix it with a `rewrite-target` annotation, or serve the app under that prefix.
15. **Ingress vs. a LoadBalancer Service?** A LoadBalancer works at L4 and needs one per Service. An Ingress works at L7, so many Services share one entry point with host/path routing and TLS.
16. **How do you enable HTTPS on an Ingress?** Create a `kubernetes.io/tls` Secret, then reference it under `spec.tls`. cert-manager can automate this.
