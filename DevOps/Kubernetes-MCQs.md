# Kubernetes: Advanced MCQs

> **Practice for the Kubernetes lectures: [8 – Basics](08-Kubernetes-Basics.md), [9 – Services & Deployments](09-Kubernetes-Objects-Services-Deployments.md), [10 – Deployment Strategies](10-Kubernetes-Deployment-Strategies.md), [11 – ConfigMaps, Secrets & Ingress](11-Kubernetes-ConfigMaps-Secrets-Ingress.md), [12 – Volumes & HPA](12-Kubernetes-Volumes-and-Autoscaling.md), [13 – Troubleshooting & Helm](13-Kubernetes-Troubleshooting-and-Helm.md) and [14 – Revision & StatefulSets](14-Kubernetes-and-Helm-Revision.md).** Each answer is shown right below its question. Cover it if you want to test yourself first.
>
> **Helm questions are in a separate file:** [Helm MCQs](Helm-MCQs.md).

| Part | Lecture | Questions |
|------|---------|-----------|
| [A](#part-a-lecture-8--architecture-pods--yaml) | 8 – Architecture, Pods & YAML | Q1–Q10 |
| [B](#part-b-lecture-9--labels-services-deployments--daemonsets) | 9 – Labels, Services, Deployments & DaemonSets | Q11–Q20 |
| [C](#part-c-lecture-10--deployment-strategies--pod-lifecycle) | 10 – Deployment Strategies & Pod Lifecycle | Q21–Q30 |
| [D](#part-d-lecture-11--configmaps-secrets--ingress) | 11 – ConfigMaps, Secrets & Ingress | Q31–Q40 |
| [E](#part-e-lecture-12--volumes--autoscaling) | 12 – Volumes & Autoscaling | Q41–Q50 |
| [F](#part-f-lectures-1314--troubleshooting--statefulsets) | 13–14 – Troubleshooting & StatefulSets | Q51–Q56 |

---

## Part A: Lecture 8 – Architecture, Pods & YAML

### Q1. Pending Pod
After `kubectl apply -f pod.yaml`, the Pod stays `Pending`. `kubectl describe` shows `0/3 nodes are available: 3 Insufficient cpu`. Which component made the decision that left the Pod Pending?
- A) kubelet
- B) kube-scheduler
- C) kube-controller-manager
- D) kube-proxy

> ✅ **Answer: B.** The scheduler filters nodes by free resources. If no node fits, the Pod stays Pending. The kubelet only runs Pods that have already been assigned to its node. · 📖 [L8 §4](08-Kubernetes-Basics.md#4-control-plane-master-node-components)

### Q2. Talking to etcd
Which component is the **only** one that reads from and writes to etcd directly?
- A) Scheduler
- B) Controller manager
- C) kube-apiserver
- D) kubelet

> ✅ **Answer: C.** Every component talks to the API server, and only the API server talks to etcd. · 📖 [L8 §3](08-Kubernetes-Basics.md#3-architecture-overview)

### Q3. Scheduler is down
The scheduler crashes, but the API server, etcd and every kubelet are healthy. You apply a new Pod manifest. What happens?
- A) The API server rejects the manifest
- B) The Pod is saved to etcd but stays `Pending` with no node assigned
- C) A kubelet picks up the Pod and runs it on its own node
- D) The Pod runs on the control plane node

> ✅ **Answer: B.** The API server checks the Pod and stores it, but nobody assigns a node. Kubelets only run Pods bound to their own node, so none of them will pick it up. · 📖 [L8 §7](08-Kubernetes-Basics.md#7-what-happens-on-kubectl-apply)

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

> ✅ **Answer: C.** The default `restartPolicy` is `Always`, so the finished container keeps being restarted. You'd see `Completed` only with `Never` (or `OnFailure`, since `echo` exits with 0). · 📖 [L8 §12](08-Kubernetes-Basics.md#12-hands-on-a-run-once-task-pod)

### Q5. Failing command with `OnFailure`
A Pod has `restartPolicy: OnFailure` and `command: ["sh", "-c", "exit 1"]`. What happens?
- A) The Pod goes to `Failed` straight away and is never restarted
- B) The container is restarted again and again with growing delays (`CrashLoopBackOff`)
- C) The Pod shows `Completed`, because the command ran
- D) The Pod stays `Pending`

> ✅ **Answer: B.** `OnFailure` restarts the container when the exit code isn't 0. `exit 1` fails every time, so you get growing restart delays. · 📖 [L8 §12](08-Kubernetes-Basics.md#12-hands-on-a-run-once-task-pod)

### Q6. A node dies
A worker node runs two Pods: **X**, a bare Pod created from a Pod manifest, and **Y**, one of the 3 replicas of a Deployment. The node loses power for good. After the node controller evicts its Pods, what happens?
- A) Both X and Y are recreated on healthy nodes
- B) Only a replacement for Y is created. X is gone for good.
- C) Both stay `Unknown` until the node comes back
- D) The kubelet on another node restarts X and Y

> ✅ **Answer: B.** A bare Pod has no controller that owns it, so nothing recreates it. Y belongs to a ReplicaSet, which notices it has only 2 replicas and creates a new Pod. The scheduler then places that Pod on a healthy node. · 📖 [L8 §6](08-Kubernetes-Basics.md#6-controllers-node--replicaset), [§13](08-Kubernetes-Basics.md#13--preview-replicasets--deployments)

### Q7. `args` only
An image was built with `ENTRYPOINT ["python"]` and `CMD ["app.py"]`. The Pod spec sets only `args: ["worker.py"]`, with no `command`. What runs?
- A) `python app.py`
- B) `python worker.py`
- C) `worker.py`
- D) `python app.py worker.py`

> ✅ **Answer: B.** `args` replaces only Docker's `CMD`. The `ENTRYPOINT` (`python`) stays, because `command` wasn't set. · 📖 [L8 §12](08-Kubernetes-Basics.md#12-hands-on-a-run-once-task-pod)

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

> ✅ **Answer: B.** For `apps/v1` Deployments, the `selector` is required and must match the template's labels. Otherwise the API server rejects the manifest. · 📖 [L8 §13](08-Kubernetes-Basics.md#13--preview-replicasets--deployments)

### Q9. Who recreates a deleted Pod?
You run `kubectl delete pod nginx-deploy-abc12` on one of the 3 Pods of a Deployment. Which controller **directly** creates the replacement Pod object?
- A) Deployment controller
- B) ReplicaSet controller
- C) Node controller
- D) Scheduler

> ✅ **Answer: B.** The Deployment manages ReplicaSets, and the ReplicaSet manages Pods. The ReplicaSet controller sees 2 Pods instead of 3 and creates one more. · 📖 [L8 §13](08-Kubernetes-Basics.md#13--preview-replicasets--deployments)

### Q10. Finding the crash output
A Pod is in `CrashLoopBackOff`. `kubectl logs <pod>` shows almost nothing, because the container has only just been restarted. Which command shows the output from the run that crashed?
- A) `kubectl describe pod <pod>`
- B) `kubectl logs <pod> --previous`
- C) `kubectl get pod <pod> -o yaml`
- D) `kubectl exec -it <pod> -- sh`

> ✅ **Answer: B.** `--previous` shows the logs of the container run that ended. `describe` shows events like "Back-off restarting", but not the app's own output. · 📖 [L8 §11](08-Kubernetes-Basics.md#11-pod-lifecycle)

---

## Part B: Lecture 9 – Labels, Services, Deployments & DaemonSets

### Q11. Rolling update limits
A Deployment has `replicas: 4`, `maxSurge: 1` and `maxUnavailable: 1`. During a rolling update, what are the **most Pods in total** and the **fewest available Pods** at any moment?
- A) 5 in total, at least 3 available
- B) 5 in total, at least 4 available
- C) 4 in total, at least 3 available
- D) 6 in total, at least 2 available

> ✅ **Answer: A.** At most `replicas + maxSurge` = 4 + 1 = **5** Pods in total, and at least `replicas − maxUnavailable` = 4 − 1 = **3** available. · 📖 [L9 §7](09-Kubernetes-Objects-Services-Deployments.md#7-rolling-updates-maxsurge--maxunavailable)

### Q12. Default percentages
A Deployment has `replicas: 10` and **no** `rollingUpdate` settings, so it uses the defaults of 25% / 25%. What are the most Pods in total and the fewest available Pods during an update?
- A) 12 in total, at least 8 available
- B) 13 in total, at least 8 available
- C) 13 in total, at least 7 available
- D) 12 in total, at least 7 available

> ✅ **Answer: B.** 25% of 10 = 2.5. Kubernetes **rounds `maxSurge` up** (3) and **`maxUnavailable` down** (2), choosing the safe side both times. So at most 10 + 3 = **13** Pods, and at least 10 − 2 = **8** available. · 📖 [L9 §7](09-Kubernetes-Objects-Services-Deployments.md#7-rolling-updates-maxsurge--maxunavailable)

### Q13. Case mismatch in a Service
A Service has `selector: { app: Web }`, but the Pods are labelled `app: web`. `kubectl apply` succeeds. What do you observe?
- A) The API server rejects the Service, like it does for a mismatched Deployment
- B) The Service is created, `kubectl get endpoints` shows `<none>`, and requests to it fail
- C) Labels aren't case-sensitive, so traffic flows normally
- D) kube-proxy relabels the Pods to match

> ✅ **Answer: B.** Labels **are** case-sensitive. A Service's selector isn't checked against existing Pods, so the Service is accepted with no endpoints and its traffic goes nowhere. Only a Deployment's selector-vs-template mismatch is rejected (Q8). · 📖 [L9 §4](09-Kubernetes-Objects-Services-Deployments.md#4-labels--selectors)

### Q14. Scaling the wrong object
The Deployment `web` has `replicas: 3`. You run `kubectl scale rs web-7c4b9 --replicas=6` on its current ReplicaSet. A few seconds later:
- A) 6 Pods keep running, and the Deployment now says 6
- B) The Deployment controller sets the ReplicaSet back to 3, and the extra Pods are removed
- C) The command fails, because a ReplicaSet owned by a Deployment can't be scaled
- D) A new ReplicaSet with 6 Pods is created next to the old one

> ✅ **Answer: B.** The Deployment owns the ReplicaSet and keeps setting it to the Deployment's `replicas` (3). Scale the Deployment instead: `kubectl scale deploy web --replicas=6`. · 📖 [L9 §5](09-Kubernetes-Objects-Services-Deployments.md#5-replicasets)

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

> ✅ **Answer: B.** The Service sends traffic to `targetPort` (8080) on the Pod. `containerPort` is mostly documentation and doesn't redirect anything. Nothing listens on 8080, so the connection is refused. · 📖 [L9 §10](09-Kubernetes-Objects-Services-Deployments.md#10-port-mapping-containerport-targetport-port-nodeport)

### Q16. NodePort on a different node
A NodePort Service uses `nodePort: 30080` in a 3-node cluster. The only matching Pod runs on **node-2**. You send a request to `node-1-IP:30080`. With default settings:
- A) The request fails, because there's no Pod on node-1
- B) It works, because kube-proxy on node-1 forwards the traffic to the Pod on node-2
- C) node-1 sends back an HTTP redirect to node-2
- D) It works only if the Pod is part of a DaemonSet

> ✅ **Answer: B.** A NodePort is opened on **every** node. kube-proxy on any node forwards to a matching Pod anywhere in the cluster. (➕ `externalTrafficPolicy: Local` would change this and send traffic only to Pods on the same node.) · 📖 [L9 §9](09-Kubernetes-Objects-Services-Deployments.md#9-service-types)

### Q17. Headless Service DNS
A Service has `clusterIP: None` and a selector matching 3 Pods. What does a DNS lookup of its name return from inside the cluster?
- A) One stable virtual IP
- B) The IP addresses of the 3 Pods
- C) The IP addresses of the nodes the Pods run on
- D) A CNAME to an external hostname

> ✅ **Answer: B.** A headless Service has no virtual IP, so DNS returns the matching Pods' IPs directly. Option D describes ExternalName. · 📖 [L9 §9](09-Kubernetes-Objects-Services-Deployments.md#9-service-types)

### Q18. A node joins
A DaemonSet runs on a 3-node cluster (`DESIRED 3`). A 4th worker node joins the cluster. What happens?
- A) Nothing, until you run `kubectl scale ds --replicas=4`
- B) A DaemonSet Pod is created on the new node automatically, and DESIRED becomes 4
- C) All 4 DaemonSet Pods are deleted and recreated
- D) You must edit `replicas: 4` in the DaemonSet YAML

> ✅ **Answer: B.** A DaemonSet has no `replicas` field. It runs one Pod per node and adds one to every new node automatically. · 📖 [L9 §11](09-Kubernetes-Objects-Services-Deployments.md#11-daemonsets)

### Q19. Undoing a rollout
You updated a Deployment from `nginx:1.14` to `nginx:1.27`, then ran `kubectl rollout undo`. What happens at the ReplicaSet level?
- A) A brand-new ReplicaSet for `nginx:1.14` is created from scratch
- B) The old `nginx:1.14` ReplicaSet (kept at 0) is scaled back up while the `1.27` ReplicaSet is scaled down
- C) The running Pods have their image changed in place, with no new Pods
- D) The Deployment is deleted and recreated from the previous YAML

> ✅ **Answer: B.** Old ReplicaSets are kept at 0 replicas. A rollback is just another rolling update that scales the old ReplicaSet back up. That's why rollbacks are fast. · 📖 [L9 §6](09-Kubernetes-Objects-Services-Deployments.md#6-deployments), [§7](09-Kubernetes-Objects-Services-Deployments.md#7-rolling-updates-maxsurge--maxunavailable)

### Q20. `LoadBalancer` on minikube
On minikube, `kubectl get svc` shows `TYPE LoadBalancer` with `EXTERNAL-IP <pending>`, and the IP never appears. What's the most likely reason?
- A) The selector doesn't match any Pods
- B) There's no cloud provider to create a real load balancer. Use `minikube tunnel` or `minikube service <name>`.
- C) `targetPort` is missing
- D) LoadBalancer Services must also be headless

> ✅ **Answer: B.** A `LoadBalancer` Service asks a cloud provider (AWS, GCP, Azure) to create a real load balancer. minikube has no cloud, so the external IP stays `<pending>`. · 📖 [L9 §9](09-Kubernetes-Objects-Services-Deployments.md#9-service-types)

---

## Part C: Lecture 10 – Deployment Strategies & Pod Lifecycle

### Q21. Green is deployed, but not switched yet
In a blue-green setup, the Service selector is `{ app: my-app, version: blue }`. You apply the green Deployment, whose Pods are labelled `{ app: my-app, version: green }`, but you **haven't** patched the Service yet. How much user traffic reaches green?
- A) 50%, because both Deployments have `app: my-app`
- B) 0%, because the Service only selects Pods labelled `version: blue`
- C) A share based on the Pod counts of blue and green
- D) 100%, because the newest Deployment takes over automatically

> ✅ **Answer: B.** A Service sends traffic only to Pods that match **all** its selector labels. Green has `version: green`, so it gets nothing until you patch the selector. That's why you can test green safely first. · 📖 [L10 §6](10-Kubernetes-Deployment-Strategies.md#6-strategy-3-blue-green)

### Q22. The canary that gets no traffic
A team runs `my-app-stable` (9 replicas, labels `app: my-app, track: stable`) and `my-app-canary` (1 replica, labels `app: my-app, track: canary`). The Service selector is `{ app: my-app, track: stable }`. Roughly what share of traffic reaches the canary?
- A) 10%
- B) 0%
- C) 50%
- D) 100%

> ✅ **Answer: B.** The selector includes `track: stable`, so canary Pods never match. For a canary to get traffic, the Service must select only the **shared** label (`app: my-app`). · 📖 [L10 §7](10-Kubernetes-Deployment-Strategies.md#7-strategy-4-canary)

### Q23. Raising the canary share
A canary currently runs as 1 canary + 9 stable Pods (≈10%) behind a Service that selects only `app`. You want **≈25%** on the canary while keeping **12 Pods in total**. Which replica counts do you set?
- A) canary 3, stable 9
- B) canary 3, stable 12
- C) canary 2, stable 10
- D) canary 4, stable 8

> ✅ **Answer: A.** With plain Services, the traffic share follows the Pod ratio: 3 / (3 + 9) = 3/12 = **25%**. B is 3/15 = 20%, C is 2/12 ≈ 17%, D is 4/12 ≈ 33%. · 📖 [L10 §7](10-Kubernetes-Deployment-Strategies.md#7-strategy-4-canary)

### Q24. Choosing a strategy
Version 2 runs a database migration that **drops a column version 1 still reads**. A 2-minute maintenance window is acceptable. Which strategy fits best?
- A) Rolling update
- B) Canary
- C) Recreate
- D) Blue-green, with both colours pointing at the same database

> ✅ **Answer: C.** v1 and v2 **can't run at the same time** (v1 would break once the column is gone). Rolling, canary and a shared-database blue-green all run both versions together. Recreate never does, and the downtime is acceptable. · 📖 [L10 §5](10-Kubernetes-Deployment-Strategies.md#5-strategy-2-recreate), [§8](10-Kubernetes-Deployment-Strategies.md#8-comparing-the-four-strategies)

### Q25. Recreate with a bad image
A Deployment with `strategy: Recreate` and 3 replicas is updated to an image tag with a typo. What happens?
- A) The old Pods keep serving until the new ones are Ready
- B) All old Pods are terminated, and the new Pods sit in `ImagePullBackOff`, so the app is completely down
- C) Kubernetes notices the bad image and pauses before terminating anything
- D) Kubernetes rolls back to the previous ReplicaSet automatically

> ✅ **Answer: B.** Recreate kills all old Pods **first**, without checking whether the new ones will work. Compare this with a rolling update with `maxUnavailable: 0`, where the old Pods would stay up. · 📖 [L10 §5](10-Kubernetes-Deployment-Strategies.md#5-strategy-2-recreate), [§9](10-Kubernetes-Deployment-Strategies.md#9-worked-example-rolling-update--verify--roll-back)

### Q26. Both limits set to 0
You apply a Deployment with `replicas: 4`, `maxSurge: 0` and `maxUnavailable: 0`. What happens?
- A) The API server rejects it with a validation error
- B) It's accepted, but any future rolling update hangs forever
- C) It's accepted and quietly behaves like `Recreate`
- D) It's accepted, and the 25% / 25% defaults are used instead

> ✅ **Answer: A.** The API server refuses `maxUnavailable: 0` when `maxSurge` is also 0, because such an update could never make progress. · 📖 [L9 §7](09-Kubernetes-Objects-Services-Deployments.md#7-rolling-updates-maxsurge--maxunavailable)

### Q27. New Pods fail their readiness probe
`replicas: 3`, `maxSurge: 1`, `maxUnavailable: 0`. The new version's `/health` endpoint returns HTTP 500, so its **readinessProbe** fails. Its livenessProbe passes. During the rollout:
- A) The new Pod is restarted again and again
- B) The rollout stalls: 3 old Pods keep serving, and 1 new Pod runs but gets no traffic
- C) Traffic is split evenly between the old and new Pods
- D) The old Pods are terminated once the new Pod is Running

> ✅ **Answer: B.** A failing **readiness** probe means "not ready for traffic". The Pod isn't restarted, just kept out of the Service. With `maxUnavailable: 0`, the old Pods are only removed once new ones are Ready, so the rollout stalls safely. After `progressDeadlineSeconds` it's marked as failed. · 📖 [L10 §2](10-Kubernetes-Deployment-Strategies.md#2-pod-lifecycle--debugging-toolkit), [§9](10-Kubernetes-Deployment-Strategies.md#9-worked-example-rolling-update--verify--roll-back)

### Q28. A failing liveness probe
A container's **livenessProbe** starts failing (the app is deadlocked). What does Kubernetes do?
- A) It removes the Pod from the Service endpoints, but leaves it running
- B) The kubelet restarts the container
- C) It deletes the Pod and schedules a new one on a different node
- D) It rolls the Deployment back to the previous revision

> ✅ **Answer: B.** A failing **liveness** probe makes the kubelet restart the container on the same node. Option A is what a failing **readiness** probe does. · 📖 [L10 §2](10-Kubernetes-Deployment-Strategies.md#2-pod-lifecycle--debugging-toolkit)

### Q29. Undo after three revisions
A Deployment went through revisions 1 (v1), 2 (v2) and 3 (v3). You run `kubectl rollout undo deploy/my-app` once. Which version runs now, and what does `kubectl rollout history` list?
- A) v2, and history lists revisions 1, 3, 4
- B) v1, and history lists revisions 1, 2, 3
- C) v2, and history lists revisions 1, 2, 3
- D) v1, and history lists revisions 2, 3, 4

> ✅ **Answer: A.** `undo` goes back **one** revision, to v2. Kubernetes reuses revision 2's ReplicaSet and gives it the new number **4**, so revision 2 disappears from the list. Running `undo` again would bring back v3, not v1. · 📖 [L10 §9](10-Kubernetes-Deployment-Strategies.md#9-worked-example-rolling-update--verify--roll-back)

### Q30. `rollout restart`
What does `kubectl rollout restart deploy/my-app` do?
- A) It replaces all Pods with fresh ones of the **same** version, using a normal rolling update
- B) It restarts the containers inside the existing Pods, keeping the Pod names
- C) It rolls back to the previous revision
- D) It deletes every Pod at once, then lets the ReplicaSet recreate them

> ✅ **Answer: A.** It adds a `restartedAt` timestamp to the Pod template. That change makes the Deployment create a new ReplicaSet and roll over to it as usual, with no downtime. It's used to reload config or Secrets without changing the image. · 📖 [L10 §9](10-Kubernetes-Deployment-Strategies.md#9-worked-example-rolling-update--verify--roll-back)

---

## Part D: Lecture 11 – ConfigMaps, Secrets & Ingress

### Q31. Plain text under `data:`
A Secret manifest contains:
```yaml
data:
  DB_PASSWORD: S3cr3t!
```
What happens on `kubectl apply`?
- A) It's stored as is, and the Pod sees `S3cr3t!`
- B) The API server rejects it, because values under `data:` must be valid base64
- C) Kubernetes base64-encodes it for you
- D) It's accepted, and the Pod sees garbled bytes

> ✅ **Answer: B.** `data:` must hold base64 strings, and `!` isn't a base64 character, so the API server refuses the manifest. To write plain text, use **`stringData:`**, and Kubernetes encodes it for you. · 📖 [L11 §5](11-Kubernetes-ConfigMaps-Secrets-Ingress.md#5-secrets)

### Q32. Updated ConfigMap, env vars
A Deployment reads a ConfigMap through `envFrom`. You edit the ConfigMap and change `LOG_LEVEL` from `info` to `debug`. What do the **running** Pods see?
- A) `debug`, within about a minute
- B) Still `info`, until the Pods are restarted (e.g. `kubectl rollout restart`)
- C) `debug`, because Kubernetes restarts the Pods automatically
- D) Still `info`, and the ConfigMap edit is rejected while Pods use it

> ✅ **Answer: B.** Environment variables are fixed when the container starts. Edits only reach new containers, so restart them with `kubectl rollout restart deploy/<name>`. · 📖 [L11 §4](11-Kubernetes-ConfigMaps-Secrets-Ingress.md#4-using-a-configmap-in-a-pod)

### Q33. Updated ConfigMap, volume mount
The same ConfigMap is **mounted as a volume** at `/etc/config`, with no `subPath`. After you edit it:
- A) The files never change while the Pod runs
- B) The files update automatically after a short delay, but the app must re-read them
- C) The Pod is restarted so it gets the new files
- D) The files update only if the Pod is deleted

> ✅ **Answer: B.** The kubelet refreshes ConfigMap volumes periodically, usually within about a minute. The app still has to notice and reload the file. (➕ `subPath` mounts are **never** updated.) · 📖 [L11 §4](11-Kubernetes-ConfigMaps-Secrets-Ingress.md#4-using-a-configmap-in-a-pod)

### Q34. Missing ConfigMap
A Pod uses `configMapKeyRef` to point at a ConfigMap called `app-config`, which **doesn't exist**. What STATUS does `kubectl get pods` show?
- A) `Pending`
- B) `CreateContainerConfigError`
- C) `CrashLoopBackOff`
- D) `Running`, with the variable empty

> ✅ **Answer: B.** The kubelet can't build the container's config without the ConfigMap, so the container is never created. Create the ConfigMap, or fix its name or key. · 📖 [L11 §4](11-Kubernetes-ConfigMaps-Secrets-Ingress.md#4-using-a-configmap-in-a-pod), [L13 §7.4](13-Kubernetes-Troubleshooting-and-Helm.md#74--more-errors-youll-meet)

### Q35. How secure is a Secret?
In a plain, self-managed cluster with default settings, which statement about Secrets is true?
- A) They're encrypted with AES in etcd by default
- B) They're only base64-encoded, so anyone allowed to `get` them can read the values. Encryption at rest must be turned on by the admin.
- C) `kubectl describe secret` shows the decoded values
- D) Mounted Secret files are written to the node's disk

> ✅ **Answer: B.** Base64 is encoding, not encryption. Real protection comes from RBAC and from encryption at rest, which the admin must turn on in a plain cluster. `describe` hides the values, and mounted Secrets live in tmpfs (RAM). · 📖 [L11 §5](11-Kubernetes-ConfigMaps-Secrets-Ingress.md#5-secrets)

### Q36. Ingress with no address
You apply an Ingress. `kubectl get ingress` shows an empty `ADDRESS`, and requests time out. The backend Service and Pods are healthy. Most likely cause?
- A) The backend Service is `ClusterIP` instead of `NodePort`
- B) No Ingress controller is installed (or `ingressClassName` points at one that doesn't exist)
- C) `pathType: Prefix` isn't supported
- D) The rule is missing `host:`

> ✅ **Answer: B.** An Ingress is only a set of rules. The controller (e.g. NGINX) is what fills in `ADDRESS` and routes traffic. Backend Services are normally plain `ClusterIP`. · 📖 [L11 §9](11-Kubernetes-ConfigMaps-Secrets-Ingress.md#9-ingress-vs-ingress-controller)

### Q37. Two matching paths
An Ingress has two rules on the same host: `/` (Prefix) → `web-svc`, and `/api` (Prefix) → `api-svc`. A request comes in for `/api/users`. Where does it go?
- A) `web-svc`, because `/` is listed first
- B) `api-svc`, because the longest matching path wins
- C) Nowhere: two rules match, so the controller returns 404
- D) Both Services receive it

> ✅ **Answer: B.** When several paths match, the **longest** match wins, whatever order the rules are listed in. · 📖 [L11 §11](11-Kubernetes-ConfigMaps-Secrets-Ingress.md#11-path-based-routing)

### Q38. Prefix matching
A rule has `path: /backend` and `pathType: Prefix`. Does a request for `/backendX/status` match it?
- A) Yes, because it starts with the characters `/backend`
- B) No, because Prefix matches whole path segments split by `/`
- C) Only if `pathType` is `Exact`
- D) It depends on the `host:` field

> ✅ **Answer: B.** `Prefix` compares whole segments, so `/backend` matches `/backend` and `/backend/...` but not `/backendX`. · 📖 [L11 §11](11-Kubernetes-ConfigMaps-Secrets-Ingress.md#11-path-based-routing)

### Q39. The path reaches the app
`myapp.local/backend/users` is routed to `backend-service`. The backend app only defines the route `/users`. With no annotations, what happens?
- A) The app receives `/users` and responds normally
- B) The app receives `/backend/users` and returns 404. Fix it with a `rewrite-target` annotation, or serve the app under `/backend`.
- C) The Ingress strips `/backend` by default
- D) The controller returns 502

> ✅ **Answer: B.** The path is forwarded unchanged. NGINX only strips it if you add `nginx.ingress.kubernetes.io/rewrite-target`. · 📖 [L11 §11](11-Kubernetes-ConfigMaps-Secrets-Ingress.md#11-path-based-routing)

### Q40. Several domains on one IP
`app1.example.com` and `app2.example.com` both resolve to the **same** IP. How does the Ingress controller decide which Service gets a request?
- A) By the destination port
- B) By the HTTP `Host` header
- C) By the client's source IP
- D) Only by the TLS certificate

> ✅ **Answer: B.** The browser sends the domain name in the `Host` header, and host-based rules match on it. (For HTTPS, the controller also uses SNI to pick the right certificate.) · 📖 [L11 §10](11-Kubernetes-ConfigMaps-Secrets-Ingress.md#10-host-based-routing)

---

## Part E: Lecture 12 – Volumes & Autoscaling

### Q41. emptyDir and a crash
A container writes files to an `emptyDir` volume, then crashes, and the kubelet restarts it in the same Pod. Are the files still there?
- A) No, because emptyDir is wiped on every container restart
- B) Yes, because emptyDir lasts as long as the **Pod**, not the container
- C) Only if `medium: Memory` is set
- D) Yes, because emptyDir is copied to a PersistentVolume

> ✅ **Answer: B.** An emptyDir is created with the Pod and removed with the Pod. A container restart doesn't touch it, which is why it's good for caches and for files shared between containers. · 📖 [L12 §3](12-Kubernetes-Volumes-and-Autoscaling.md#3-emptydir-scratch-space-for-a-pod)

### Q42. hostPath on another node
A Deployment's Pod writes to `hostPath: /mnt/data` on node-1. The Pod is deleted, and its replacement is scheduled on **node-2**. What does it see in `/mnt/data`?
- A) The same files, because hostPath follows the Pod
- B) node-2's own `/mnt/data`, without the files written on node-1
- C) Files copied over from node-1 automatically
- D) An error, because hostPath Pods can't move

> ✅ **Answer: B.** hostPath is a folder on whichever node the Pod lands on. The data stays on node-1. That's why hostPath isn't used for app data in production. · 📖 [L12 §4](12-Kubernetes-Volumes-and-Autoscaling.md#4-hostpath-a-folder-from-the-node)

### Q43. Claim smaller than the volume
A static PV of **1Gi** binds to a PVC that requests **500Mi**. What CAPACITY does `kubectl get pvc` show?
- A) 500Mi
- B) 1Gi
- C) 524Mi
- D) Nothing, until a Pod mounts it

> ✅ **Answer: B.** Binding is one-to-one, so the claim gets the **whole** PV. The extra 524Mi can't be used by anyone else. · 📖 [L12 §6](12-Kubernetes-Volumes-and-Autoscaling.md#6-static-provisioning-write-the-pv-yourself)

### Q44. The default StorageClass
On minikube (default StorageClass `standard`), you create a static PV and a PVC, and **neither** sets `storageClassName`. What happens?
- A) The PVC binds to your static PV
- B) The PVC is given the `standard` class, a new PV is created for it, and your static PV stays `Available`
- C) The PVC stays `Pending` forever
- D) The API server rejects the PVC

> ✅ **Answer: B.** A PVC with no `storageClassName` gets the default class, so it's provisioned dynamically. The static PV has no class, so they never match. Set `storageClassName: ""` on **both** to bind them. · 📖 [L12 §6](12-Kubernetes-Volumes-and-Autoscaling.md#6-static-provisioning-write-the-pv-yourself)

### Q45. Three replicas, one RWO disk
A Deployment has 3 replicas spread over 3 nodes, and they all mount the **same** PVC, backed by an AWS EBS disk with `ReadWriteOnce`. What happens?
- A) All 3 Pods run and share the disk
- B) Pods on other nodes get stuck in `ContainerCreating` with a Multi-Attach error. Shared storage needs RWX, such as EFS or NFS.
- C) RWO means one Pod, so Kubernetes runs only 1 replica
- D) Kubernetes copies the data to each node

> ✅ **Answer: B.** RWO means **one node** at a time, not one Pod. Replicas on that same node would work, but others can't attach the disk. Shared read-write storage needs RWX, which block disks like EBS can't do. · 📖 [L12 §8](12-Kubernetes-Volumes-and-Autoscaling.md#8-access-modes-reclaim-policies--pv-phases)

### Q46. Deleting the claim
A PV has `persistentVolumeReclaimPolicy: Retain`. Its PVC is deleted. What is the PV's status?
- A) `Available`, ready to be reused straight away
- B) `Released`: the data is kept, and the PV needs manual cleanup before it can be reused
- C) It's deleted, along with the disk
- D) It stays `Bound`

> ✅ **Answer: B.** `Retain` protects the data. The PV becomes `Released` and still records the old claim, so an admin must clean it up (or delete and recreate it) before reuse. · 📖 [L12 §8](12-Kubernetes-Volumes-and-Autoscaling.md#8-access-modes-reclaim-policies--pv-phases)

### Q47. HPA shows `<unknown>`
An HPA shows `TARGETS <unknown>/50%`. metrics-server is running and `kubectl top pods` prints numbers. What's most likely missing?
- A) `maxReplicas`
- B) `resources.requests.cpu` on the containers
- C) A livenessProbe
- D) A Service in front of the Deployment

> ✅ **Answer: B.** `kubectl top` works without requests, but HPA utilization is usage ÷ **request**. With no CPU request there's nothing to divide by, so the target is `<unknown>`. · 📖 [L12 §10](12-Kubernetes-Volumes-and-Autoscaling.md#10-horizontal-pod-autoscaler-hpa)

### Q48. HPA calculation
An HPA targets **60%** CPU with `maxReplicas: 5`. There are **3** replicas, averaging **120%** of their CPU request. How many replicas will it set?
- A) 6
- B) 5
- C) 4
- D) 3

> ✅ **Answer: B.** ceil(3 × 120 / 60) = **6**, but `maxReplicas` caps it at **5**. · 📖 [L12 §10](12-Kubernetes-Volumes-and-Autoscaling.md#10-horizontal-pod-autoscaler-hpa)

### Q49. Re-applying the Deployment
An HPA has scaled a Deployment to 6 replicas. You re-apply the Deployment YAML, which still contains `replicas: 2`. What happens?
- A) The HPA's `minReplicas` changes to 2
- B) The Deployment drops to 2, then the HPA scales it back up. Remove `replicas:` from YAML that an HPA manages.
- C) `kubectl apply` is rejected while an HPA exists
- D) The HPA is deleted

> ✅ **Answer: B.** `apply` sets `spec.replicas` back to 2, and the HPA then raises it again, so the two keep fighting. Once an HPA owns the count, leave `replicas:` out of the YAML. · 📖 [L12 §10](12-Kubernetes-Volumes-and-Autoscaling.md#10-horizontal-pod-autoscaler-hpa)

### Q50. Load drops to zero
Traffic stops completely. When does the HPA (default settings) start removing Pods?
- A) Immediately, at the next 15-second check
- B) After the scale-down stabilization window, which is 5 minutes by default
- C) Never: an HPA only scales up
- D) Instantly down to `minReplicas`, without waiting

> ✅ **Answer: B.** Scale-up is fast, but scale-down waits through a 5-minute stabilization window by default, to avoid flapping on spiky traffic. · 📖 [L12 §10](12-Kubernetes-Volumes-and-Autoscaling.md#10-horizontal-pod-autoscaler-hpa)

---

## Part F: Lectures 13–14 – Troubleshooting & StatefulSets

### Q51. Pending for memory
A Pod is `Pending` with `0/3 nodes are available: 3 Insufficient memory`. Its container sets `requests.memory: 8Gi` and `limits.memory: 16Gi`. Which change can get it scheduled?
- A) Raise `limits.memory` to 32Gi
- B) Lower `requests.memory` (or add a node with more free memory)
- C) Add a livenessProbe
- D) Set `restartPolicy: Never`

> ✅ **Answer: B.** The scheduler only looks at **requests** when placing Pods. Limits don't matter, so raising them changes nothing. · 📖 [L13 §7.3](13-Kubernetes-Troubleshooting-and-Helm.md#73-pending-the-pod-isnt-placed-on-any-node)

### Q52. OOMKilled
`kubectl describe pod` shows `Last State: Terminated, Reason: OOMKilled, Exit Code: 137`, and the Pod is in `CrashLoopBackOff`. What's the right fix?
- A) Raise `requests.cpu`
- B) Raise `limits.memory`, or fix the app's memory leak
- C) Change the image tag
- D) Add a readinessProbe

> ✅ **Answer: B.** OOMKilled (exit 137) means the container went over its **memory limit**. CPU settings don't affect this. · 📖 [L13 §7.1](13-Kubernetes-Troubleshooting-and-Helm.md#71-crashloopbackoff-the-app-starts-then-dies-again-and-again)

### Q53. No logs
A Pod is in `ImagePullBackOff`, and `kubectl logs <pod>` returns an error instead of output. Why, and where should you look?
- A) Logs are disabled for that namespace. Enable them.
- B) The container never started, so there's no app output. Read the Events in `kubectl describe pod`.
- C) You need `--previous`
- D) The logs went to the node's syslog. SSH into the node.

> ✅ **Answer: B.** No container was ever started, so there's nothing to log. The pull error is in the Events at the bottom of `describe`. · 📖 [L13 §4](13-Kubernetes-Troubleshooting-and-Helm.md#4-kubectl-logs-what-did-the-app-say), [§3](13-Kubernetes-Troubleshooting-and-Helm.md#3-kubectl-describe-why-is-it-in-that-state)

### Q54. No shell in the image
`kubectl exec -it api-7f9c -- bash` fails with `executable file not found in $PATH`. The image is **distroless**. What's the best way to inspect the running container?
- A) `kubectl exec -it api-7f9c -- apt install bash`
- B) `kubectl debug -it api-7f9c --image=busybox --target=<container>`
- C) Delete the Pod so it restarts with a shell
- D) `kubectl logs api-7f9c --previous`

> ✅ **Answer: B.** Distroless images have no shell and no package manager, so A can't work. `kubectl debug` attaches a temporary container that has tools, and `--target` lets it see the app's processes. · 📖 [L13 §5](13-Kubernetes-Troubleshooting-and-Helm.md#5-kubectl-exec-look-inside)

### Q55. A StatefulSet Pod is deleted
A StatefulSet `db` (3 replicas, with `volumeClaimTemplates` named `data`) is running. You delete Pod `db-1`. What happens?
- A) A Pod with a random name is created, with a new empty PVC
- B) A new `db-1` is created and re-attaches PVC `data-db-1`, so the data is kept
- C) A Pod called `db-3` is created to replace it
- D) `db-1`'s data is lost, but `db-0` and `db-2` keep theirs

> ✅ **Answer: B.** StatefulSet Pods keep a stable identity. The replacement has the same name and re-attaches the same PVC. · 📖 [L14 §2](14-Kubernetes-and-Helm-Revision.md#2-statefulsets-new)

### Q56. Scale down, then up
You scale that StatefulSet from 3 to 1, then later back to 3. What about the data that `db-1` and `db-2` had?
- A) It's gone, because their PVCs were deleted on scale-down
- B) It's back: PVCs `data-db-1` and `data-db-2` were kept and re-attach to the new `db-1` and `db-2`
- C) It was moved into `db-0`'s volume on scale-down
- D) It depends only on the StorageClass's reclaim policy

> ✅ **Answer: B.** By default, scaling down or deleting a StatefulSet **doesn't delete its PVCs**. When the Pods come back with the same names, they re-attach their old volumes. · 📖 [L14 §2](14-Kubernetes-and-Helm-Revision.md#2-statefulsets-new)

