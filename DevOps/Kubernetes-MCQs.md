# Kubernetes: Advanced MCQs

> **Practice for [Lecture 8 – Kubernetes Basics](08-Kubernetes-Basics.md), [Lecture 9 – Labels, Services, Deployments & DaemonSets](09-Kubernetes-Objects-Services-Deployments.md) and [Lecture 10 – Deployment Strategies](10-Kubernetes-Deployment-Strategies.md).** Try each part first, then open its answer key.

| Part | Lecture | Questions |
|------|---------|-----------|
| [A](#part-a-lecture-8--architecture-pods--yaml) | 8 – Architecture, Pods & YAML | Q1–Q10 |
| [B](#part-b-lecture-9--labels-services-deployments--daemonsets) | 9 – Labels, Services, Deployments & DaemonSets | Q11–Q20 |
| [C](#part-c-lecture-10--deployment-strategies--pod-lifecycle) | 10 – Deployment Strategies & Pod Lifecycle | Q21–Q30 |

---

## Part A: Lecture 8 – Architecture, Pods & YAML

### Q1. Pending Pod
After `kubectl apply -f pod.yaml`, the Pod stays `Pending`. `kubectl describe` shows `0/3 nodes are available: 3 Insufficient cpu`. Which component made the decision that left the Pod Pending?
- A) kubelet
- B) kube-scheduler
- C) kube-controller-manager
- D) kube-proxy

### Q2. Talking to etcd
Which component is the **only** one that reads from and writes to etcd directly?
- A) Scheduler
- B) Controller manager
- C) kube-apiserver
- D) kubelet

### Q3. Scheduler is down
The scheduler crashes, but the API server, etcd and every kubelet are healthy. You apply a new Pod manifest. What happens?
- A) The API server rejects the manifest
- B) The Pod is saved to etcd but stays `Pending` with no node assigned
- C) A kubelet picks up the Pod and runs it on its own node
- D) The Pod runs on the control plane node

### Q4. The run-once Pod
```yaml
spec:
  containers:
    - name: c
      image: busybox
      command: ["echo", "Hello Kubernetes"]
```
There is no `restartPolicy` set. What does `kubectl get pods` most likely show after a minute or two?
- A) `Completed`, with 0 restarts
- B) `Succeeded`, with 0 restarts
- C) `CrashLoopBackOff`, with a growing RESTARTS count
- D) `Failed`

### Q5. Failing command with `OnFailure`
A Pod has `restartPolicy: OnFailure` and `command: ["sh", "-c", "exit 1"]`. What happens?
- A) The Pod goes to `Failed` straight away and is never restarted
- B) The container is restarted again and again with growing delays (`CrashLoopBackOff`)
- C) The Pod shows `Completed`, because the command ran
- D) The Pod stays `Pending`

### Q6. A node dies
A worker node runs two Pods: **X**, a bare Pod created from a Pod manifest, and **Y**, one of the 3 replicas of a Deployment. The node loses power for good. After the node controller evicts its Pods, what happens?
- A) Both X and Y are recreated on healthy nodes
- B) Only a replacement for Y is created. X is gone for good.
- C) Both stay `Unknown` until the node comes back
- D) The kubelet on another node restarts X and Y

### Q7. `args` only
An image was built with `ENTRYPOINT ["python"]` and `CMD ["app.py"]`. The Pod spec sets only `args: ["worker.py"]`, with no `command`. What runs?
- A) `python app.py`
- B) `python worker.py`
- C) `worker.py`
- D) `python app.py worker.py`

### Q8. Label mismatch
```yaml
apiVersion: apps/v1
kind: Deployment
spec:
  replicas: 3
  selector:
    matchLabels: { app: nginx }
  template:
    metadata:
      labels: { app: web }
    ...
```
What happens on `kubectl apply`?
- A) The Deployment is created, but its Pods aren't managed by it
- B) The API server rejects it because the selector doesn't match the template labels
- C) The ReplicaSet creates new Pods forever, because it never sees one that matches
- D) It works, because `selector` is optional

### Q9. Who recreates a deleted Pod?
You run `kubectl delete pod nginx-deploy-abc12` on one of the 3 Pods of a Deployment. Which controller **directly** creates the replacement Pod object?
- A) Deployment controller
- B) ReplicaSet controller
- C) Node controller
- D) Scheduler

### Q10. Finding the crash output
A Pod is in `CrashLoopBackOff`. `kubectl logs <pod>` shows almost nothing, because the container has only just been restarted. Which command shows the output from the run that crashed?
- A) `kubectl describe pod <pod>`
- B) `kubectl logs <pod> --previous`
- C) `kubectl get pod <pod> -o yaml`
- D) `kubectl exec -it <pod> -- sh`

<details>
<summary><b>Answer Key: Part A</b> (click to open)</summary>

| Q | Ans | Why | Notes |
|---|-----|-----|-------|
| 1 | **B** | The scheduler filters nodes by free resources. If no node fits, the Pod stays Pending. The kubelet only runs Pods that have already been assigned to its node. | [L8 §4](08-Kubernetes-Basics.md#4-control-plane-master-node-components) |
| 2 | **C** | Every component talks to the API server, and only the API server talks to etcd. | [L8 §3](08-Kubernetes-Basics.md#3-architecture-overview) |
| 3 | **B** | The API server checks the Pod and stores it, but nobody assigns a node. Kubelets only run Pods bound to their own node, so none of them will pick it up. | [L8 §7](08-Kubernetes-Basics.md#7-what-happens-on-kubectl-apply) |
| 4 | **C** | The default `restartPolicy` is `Always`, so the finished container keeps being restarted. You'd see `Completed` only with `Never` (or `OnFailure`, since `echo` exits with 0). | [L8 §12](08-Kubernetes-Basics.md#12-hands-on-a-run-once-task-pod) |
| 5 | **B** | `OnFailure` restarts the container when the exit code isn't 0. `exit 1` fails every time, so you get growing restart delays. | [L8 §12](08-Kubernetes-Basics.md#12-hands-on-a-run-once-task-pod) |
| 6 | **B** | A bare Pod has no controller that owns it, so nothing recreates it. Y belongs to a ReplicaSet, which notices it has only 2 replicas and creates a new Pod. The scheduler then places that Pod on a healthy node. | [L8 §6](08-Kubernetes-Basics.md#6-controllers-node--replicaset), [§13](08-Kubernetes-Basics.md#13--preview-replicasets--deployments) |
| 7 | **B** | `args` replaces only Docker's `CMD`. The `ENTRYPOINT` (`python`) stays, because `command` wasn't set. | [L8 §12](08-Kubernetes-Basics.md#12-hands-on-a-run-once-task-pod) |
| 8 | **B** | For `apps/v1` Deployments, the `selector` is required and must match the template's labels. Otherwise the API server rejects the manifest. | [L8 §13](08-Kubernetes-Basics.md#13--preview-replicasets--deployments) |
| 9 | **B** | The Deployment manages ReplicaSets, and the ReplicaSet manages Pods. The ReplicaSet controller sees 2 Pods instead of 3 and creates one more. | [L8 §13](08-Kubernetes-Basics.md#13--preview-replicasets--deployments) |
| 10 | **B** | `--previous` shows the logs of the container run that ended. `describe` shows events like "Back-off restarting", but not the app's own output. | [L8 §11](08-Kubernetes-Basics.md#11-pod-lifecycle) |

</details>

---

## Part B: Lecture 9 – Labels, Services, Deployments & DaemonSets

### Q11. Rolling update limits
A Deployment has `replicas: 4`, `maxSurge: 1` and `maxUnavailable: 1`. During a rolling update, what are the **most Pods in total** and the **fewest available Pods** at any moment?
- A) 5 in total, at least 3 available
- B) 5 in total, at least 4 available
- C) 4 in total, at least 3 available
- D) 6 in total, at least 2 available

### Q12. Default percentages
A Deployment has `replicas: 10` and **no** `rollingUpdate` settings, so it uses the defaults of 25% / 25%. What are the most Pods in total and the fewest available Pods during an update?
- A) 12 in total, at least 8 available
- B) 13 in total, at least 8 available
- C) 13 in total, at least 7 available
- D) 12 in total, at least 7 available

### Q13. Case mismatch in a Service
A Service has `selector: { app: Web }`, but the Pods are labelled `app: web`. `kubectl apply` succeeds. What do you observe?
- A) The API server rejects the Service, like it does for a mismatched Deployment
- B) The Service is created, `kubectl get endpoints` shows `<none>`, and requests to it fail
- C) Labels aren't case-sensitive, so traffic flows normally
- D) kube-proxy relabels the Pods to match

### Q14. Scaling the wrong object
The Deployment `web` has `replicas: 3`. You run `kubectl scale rs web-7c4b9 --replicas=6` on its current ReplicaSet. A few seconds later:
- A) 6 Pods keep running, and the Deployment now says 6
- B) The Deployment controller sets the ReplicaSet back to 3, and the extra Pods are removed
- C) The command fails, because a ReplicaSet owned by a Deployment can't be scaled
- D) A new ReplicaSet with 6 Pods is created next to the old one

### Q15. Wrong `targetPort`
```yaml
# Service                       # Pod
ports:                          containers:
  - port: 80                      - image: nginx      # listens on 80
    targetPort: 8080                ports:
                                      - containerPort: 80
```
Another Pod runs `curl http://backend:80`. What happens?
- A) It works, because `containerPort: 80` tells the Service where to send traffic
- B) The connection is refused, because the Service forwards to port 8080 on the Pod and nothing listens there
- C) DNS fails to resolve `backend`
- D) It works, because kube-proxy notices the mismatch and corrects it

### Q16. NodePort on a different node
A NodePort Service uses `nodePort: 30080` in a 3-node cluster. The only matching Pod runs on **node-2**. You send a request to `node-1-IP:30080`. With default settings:
- A) The request fails, because there's no Pod on node-1
- B) It works, because kube-proxy on node-1 forwards the traffic to the Pod on node-2
- C) node-1 sends back an HTTP redirect to node-2
- D) It works only if the Pod is part of a DaemonSet

### Q17. Headless Service DNS
A Service has `clusterIP: None` and a selector matching 3 Pods. What does a DNS lookup of its name return from inside the cluster?
- A) One stable virtual IP
- B) The IP addresses of the 3 Pods
- C) The IP addresses of the nodes the Pods run on
- D) A CNAME to an external hostname

### Q18. A node joins
A DaemonSet runs on a 3-node cluster (`DESIRED 3`). A 4th worker node joins the cluster. What happens?
- A) Nothing, until you run `kubectl scale ds --replicas=4`
- B) A DaemonSet Pod is created on the new node automatically, and DESIRED becomes 4
- C) All 4 DaemonSet Pods are deleted and recreated
- D) You must edit `replicas: 4` in the DaemonSet YAML

### Q19. Undoing a rollout
You updated a Deployment from `nginx:1.14` to `nginx:1.27`, then ran `kubectl rollout undo`. What happens at the ReplicaSet level?
- A) A brand-new ReplicaSet for `nginx:1.14` is created from scratch
- B) The old `nginx:1.14` ReplicaSet (kept at 0) is scaled back up while the `1.27` ReplicaSet is scaled down
- C) The running Pods have their image changed in place, with no new Pods
- D) The Deployment is deleted and recreated from the previous YAML

### Q20. `LoadBalancer` on minikube
On minikube, `kubectl get svc` shows `TYPE LoadBalancer` with `EXTERNAL-IP <pending>`, and the IP never appears. What's the most likely reason?
- A) The selector doesn't match any Pods
- B) There's no cloud provider to create a real load balancer. Use `minikube tunnel` or `minikube service <name>`.
- C) `targetPort` is missing
- D) LoadBalancer Services must also be headless

<details>
<summary><b>Answer Key: Part B</b> (click to open)</summary>

| Q | Ans | Why | Notes |
|---|-----|-----|-------|
| 11 | **A** | At most `replicas + maxSurge` = 4 + 1 = **5** Pods in total, and at least `replicas − maxUnavailable` = 4 − 1 = **3** available. | [L9 §7](09-Kubernetes-Objects-Services-Deployments.md#7-rolling-updates-maxsurge--maxunavailable) |
| 12 | **B** | 25% of 10 = 2.5. Kubernetes **rounds `maxSurge` up** (3) and **`maxUnavailable` down** (2), choosing the safe side both times. So at most 10 + 3 = **13** Pods, and at least 10 − 2 = **8** available. | [L9 §7](09-Kubernetes-Objects-Services-Deployments.md#7-rolling-updates-maxsurge--maxunavailable) |
| 13 | **B** | Labels **are** case-sensitive. A Service's selector isn't checked against existing Pods, so the Service is accepted with no endpoints and its traffic goes nowhere. Only a Deployment's selector-vs-template mismatch is rejected (Q8). | [L9 §4](09-Kubernetes-Objects-Services-Deployments.md#4-labels--selectors) |
| 14 | **B** | The Deployment owns the ReplicaSet and keeps setting it to the Deployment's `replicas` (3). Scale the Deployment instead: `kubectl scale deploy web --replicas=6`. | [L9 §5](09-Kubernetes-Objects-Services-Deployments.md#5-replicasets) |
| 15 | **B** | The Service sends traffic to `targetPort` (8080) on the Pod. `containerPort` is mostly documentation and doesn't redirect anything. Nothing listens on 8080, so the connection is refused. | [L9 §10](09-Kubernetes-Objects-Services-Deployments.md#10-port-mapping-containerport-targetport-port-nodeport) |
| 16 | **B** | A NodePort is opened on **every** node. kube-proxy on any node forwards to a matching Pod anywhere in the cluster. (➕ `externalTrafficPolicy: Local` would change this and send traffic only to Pods on the same node.) | [L9 §9](09-Kubernetes-Objects-Services-Deployments.md#9-service-types) |
| 17 | **B** | A headless Service has no virtual IP, so DNS returns the matching Pods' IPs directly. Option D describes ExternalName. | [L9 §9](09-Kubernetes-Objects-Services-Deployments.md#9-service-types) |
| 18 | **B** | A DaemonSet has no `replicas` field. It runs one Pod per node and adds one to every new node automatically. | [L9 §11](09-Kubernetes-Objects-Services-Deployments.md#11-daemonsets) |
| 19 | **B** | Old ReplicaSets are kept at 0 replicas. A rollback is just another rolling update that scales the old ReplicaSet back up. That's why rollbacks are fast. | [L9 §6](09-Kubernetes-Objects-Services-Deployments.md#6-deployments), [§7](09-Kubernetes-Objects-Services-Deployments.md#7-rolling-updates-maxsurge--maxunavailable) |
| 20 | **B** | A `LoadBalancer` Service asks a cloud provider (AWS, GCP, Azure) to create a real load balancer. minikube has no cloud, so the external IP stays `<pending>`. | [L9 §9](09-Kubernetes-Objects-Services-Deployments.md#9-service-types) |

</details>

---

## Part C: Lecture 10 – Deployment Strategies & Pod Lifecycle

### Q21. Green is deployed, but not switched yet
In a blue-green setup, the Service selector is `{ app: my-app, version: blue }`. You apply the green Deployment, whose Pods are labelled `{ app: my-app, version: green }`, but you **haven't** patched the Service yet. How much user traffic reaches green?
- A) 50%, because both Deployments have `app: my-app`
- B) 0%, because the Service only selects Pods labelled `version: blue`
- C) A share based on the Pod counts of blue and green
- D) 100%, because the newest Deployment takes over automatically

### Q22. The canary that gets no traffic
A team runs `my-app-stable` (9 replicas, labels `app: my-app, track: stable`) and `my-app-canary` (1 replica, labels `app: my-app, track: canary`). The Service selector is `{ app: my-app, track: stable }`. Roughly what share of traffic reaches the canary?
- A) 10%
- B) 0%
- C) 50%
- D) 100%

### Q23. Raising the canary share
A canary currently runs as 1 canary + 9 stable Pods (≈10%) behind a Service that selects only `app`. You want **≈25%** on the canary while keeping **12 Pods in total**. Which replica counts do you set?
- A) canary 3, stable 9
- B) canary 3, stable 12
- C) canary 2, stable 10
- D) canary 4, stable 8

### Q24. Choosing a strategy
Version 2 runs a database migration that **drops a column version 1 still reads**. A 2-minute maintenance window is acceptable. Which strategy fits best?
- A) Rolling update
- B) Canary
- C) Recreate
- D) Blue-green, with both colours pointing at the same database

### Q25. Recreate with a bad image
A Deployment with `strategy: Recreate` and 3 replicas is updated to an image tag with a typo. What happens?
- A) The old Pods keep serving until the new ones are Ready
- B) All old Pods are terminated, and the new Pods sit in `ImagePullBackOff`, so the app is completely down
- C) Kubernetes notices the bad image and pauses before terminating anything
- D) Kubernetes rolls back to the previous ReplicaSet automatically

### Q26. Both limits set to 0
You apply a Deployment with `replicas: 4`, `maxSurge: 0` and `maxUnavailable: 0`. What happens?
- A) The API server rejects it with a validation error
- B) It's accepted, but any future rolling update hangs forever
- C) It's accepted and quietly behaves like `Recreate`
- D) It's accepted, and the 25% / 25% defaults are used instead

### Q27. New Pods fail their readiness probe
`replicas: 3`, `maxSurge: 1`, `maxUnavailable: 0`. The new version's `/health` endpoint returns HTTP 500, so its **readinessProbe** fails. Its livenessProbe passes. During the rollout:
- A) The new Pod is restarted again and again
- B) The rollout stalls: 3 old Pods keep serving, and 1 new Pod runs but gets no traffic
- C) Traffic is split evenly between the old and new Pods
- D) The old Pods are terminated once the new Pod is Running

### Q28. A failing liveness probe
A container's **livenessProbe** starts failing (the app is deadlocked). What does Kubernetes do?
- A) It removes the Pod from the Service endpoints, but leaves it running
- B) The kubelet restarts the container
- C) It deletes the Pod and schedules a new one on a different node
- D) It rolls the Deployment back to the previous revision

### Q29. Undo after three revisions
A Deployment went through revisions 1 (v1), 2 (v2) and 3 (v3). You run `kubectl rollout undo deploy/my-app` once. Which version runs now, and what does `kubectl rollout history` list?
- A) v2, and history lists revisions 1, 3, 4
- B) v1, and history lists revisions 1, 2, 3
- C) v2, and history lists revisions 1, 2, 3
- D) v1, and history lists revisions 2, 3, 4

### Q30. `rollout restart`
What does `kubectl rollout restart deploy/my-app` do?
- A) It replaces all Pods with fresh ones of the **same** version, using a normal rolling update
- B) It restarts the containers inside the existing Pods, keeping the Pod names
- C) It rolls back to the previous revision
- D) It deletes every Pod at once, then lets the ReplicaSet recreate them

<details>
<summary><b>Answer Key: Part C</b> (click to open)</summary>

| Q | Ans | Why | Notes |
|---|-----|-----|-------|
| 21 | **B** | A Service sends traffic only to Pods that match **all** its selector labels. Green has `version: green`, so it gets nothing until you patch the selector. That's why you can test green safely first. | [L10 §6](10-Kubernetes-Deployment-Strategies.md#6-strategy-3-blue-green) |
| 22 | **B** | The selector includes `track: stable`, so canary Pods never match. For a canary to get traffic, the Service must select only the **shared** label (`app: my-app`). | [L10 §7](10-Kubernetes-Deployment-Strategies.md#7-strategy-4-canary) |
| 23 | **A** | With plain Services, the traffic share follows the Pod ratio: 3 / (3 + 9) = 3/12 = **25%**. B is 3/15 = 20%, C is 2/12 ≈ 17%, D is 4/12 ≈ 33%. | [L10 §7](10-Kubernetes-Deployment-Strategies.md#7-strategy-4-canary) |
| 24 | **C** | v1 and v2 **can't run at the same time** (v1 would break once the column is gone). Rolling, canary and a shared-database blue-green all run both versions together. Recreate never does, and the downtime is acceptable. | [L10 §5](10-Kubernetes-Deployment-Strategies.md#5-strategy-2-recreate), [§8](10-Kubernetes-Deployment-Strategies.md#8-comparing-the-four-strategies) |
| 25 | **B** | Recreate kills all old Pods **first**, without checking whether the new ones will work. Compare this with a rolling update with `maxUnavailable: 0`, where the old Pods would stay up. | [L10 §5](10-Kubernetes-Deployment-Strategies.md#5-strategy-2-recreate), [§9](10-Kubernetes-Deployment-Strategies.md#9-worked-example-rolling-update--verify--roll-back) |
| 26 | **A** | The API server refuses `maxUnavailable: 0` when `maxSurge` is also 0, because such an update could never make progress. | [L9 §7](09-Kubernetes-Objects-Services-Deployments.md#7-rolling-updates-maxsurge--maxunavailable) |
| 27 | **B** | A failing **readiness** probe means "not ready for traffic". The Pod isn't restarted, just kept out of the Service. With `maxUnavailable: 0`, the old Pods are only removed once new ones are Ready, so the rollout stalls safely. After `progressDeadlineSeconds` it's marked as failed. | [L10 §2](10-Kubernetes-Deployment-Strategies.md#2-pod-lifecycle--debugging-toolkit), [§9](10-Kubernetes-Deployment-Strategies.md#9-worked-example-rolling-update--verify--roll-back) |
| 28 | **B** | A failing **liveness** probe makes the kubelet restart the container on the same node. Option A is what a failing **readiness** probe does. | [L10 §2](10-Kubernetes-Deployment-Strategies.md#2-pod-lifecycle--debugging-toolkit) |
| 29 | **A** | `undo` goes back **one** revision, to v2. Kubernetes reuses revision 2's ReplicaSet and gives it the new number **4**, so revision 2 disappears from the list. Running `undo` again would bring back v3, not v1. | [L10 §9](10-Kubernetes-Deployment-Strategies.md#9-worked-example-rolling-update--verify--roll-back) |
| 30 | **A** | It adds a `restartedAt` timestamp to the Pod template. That change makes the Deployment create a new ReplicaSet and roll over to it as usual, with no downtime. It's used to reload config or Secrets without changing the image. | [L10 §9](10-Kubernetes-Deployment-Strategies.md#9-worked-example-rolling-update--verify--roll-back) |

</details>
