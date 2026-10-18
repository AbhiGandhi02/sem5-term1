# Helm: Advanced MCQs

> **Practice for [Lecture 13 – Troubleshooting & Helm](13-Kubernetes-Troubleshooting-and-Helm.md#10-helm-why-a-package-manager) and [Lecture 14 – Kubernetes & Helm Revision](14-Kubernetes-and-Helm-Revision.md#7-helm-commands).** Each answer is shown right below its question. Cover it if you want to test yourself first.
>
> **Kubernetes questions:** [Kubernetes MCQs](Kubernetes-MCQs.md).

| Part | Topic | Questions |
|------|-------|-----------|
| [A](#part-a-concepts--charts) | Concepts & charts | Q1–Q6 |
| [B](#part-b-values--templates) | Values & templates | Q7–Q11 |
| [C](#part-c-release-lifecycle) | Release lifecycle | Q12–Q15 |

---

## Part A: Concepts & Charts

### Q1. Charts vs. releases
You run `helm install shop-dev ./shop-chart -n dev` and `helm install shop-prod ./shop-chart -n prod`. How many **charts** and **releases** are there?
- A) 2 charts, 2 releases
- B) 1 chart, 2 releases
- C) 1 chart, 1 release with 2 revisions
- D) 2 charts, 1 release

> ✅ **Answer: B.** A **chart** is the package. A **release** is one installed copy of it, with its own name. The same chart can be installed many times. Revisions only appear when one release is upgraded or rolled back. · 📖 [L13 §11](13-Kubernetes-Troubleshooting-and-Helm.md#11-helm-key-terms)

### Q2. Repo not found
`helm install my-nginx bitnami/nginx` fails with `Error: repo bitnami not found`. What's the fix?
- A) `helm create bitnami`
- B) `helm repo add bitnami https://charts.bitnami.com/bitnami`, then `helm repo update`
- C) `kubectl create namespace bitnami`
- D) `helm pull nginx`

> ✅ **Answer: B.** `bitnami/nginx` means "chart `nginx` from the repo I named `bitnami`". Helm only knows repos you've added. (➕ Many charts can also be installed straight from an OCI registry with no `repo add`.) · 📖 [L13 §12](13-Kubernetes-Troubleshooting-and-Helm.md#12-helm-basic-commands)

### Q3. `version` vs. `appVersion`
You add a readinessProbe to `templates/deployment.yaml`. The application image doesn't change. What should you update in `Chart.yaml`?
- A) `appVersion` only
- B) `version` only
- C) Both `version` and `appVersion`
- D) Neither: Chart.yaml only matters on the first install

> ✅ **Answer: B.** `version` is the **chart package's** version, so bump it whenever the templates or defaults change. `appVersion` is the version of the **app** being deployed (often the default image tag), and only changes with the app. · 📖 [L13 §13](13-Kubernetes-Troubleshooting-and-Helm.md#13-creating-a-custom-chart)

### Q4. The underscore file
Why is the file called `templates/_helpers.tpl`, with a leading underscore?
- A) It makes the file hidden from `helm lint`
- B) Files starting with `_` aren't rendered as Kubernetes manifests. They hold reusable snippets (names, labels) for other templates.
- C) It marks the file as optional
- D) Helm only reads files that start with `_`

> ✅ **Answer: B.** Helm renders every file in `templates/` into a manifest **except** those starting with `_`. `_helpers.tpl` defines named templates that other files include. · 📖 [L13 §13](13-Kubernetes-Troubleshooting-and-Helm.md#13-creating-a-custom-chart)

### Q5. Where Helm 3 keeps its history
Where does **Helm 3** store a release's history (revisions and their values)?
- A) In a server-side component called Tiller
- B) In Secrets in the release's namespace
- C) In a `.helm/` folder on your laptop only
- D) In etcd, through a special Helm controller Pod

> ✅ **Answer: B.** Helm 3 is client-only. Each revision is saved as a Secret named like `sh.helm.release.v1.<release>.v<N>`, so anyone with cluster access sees the same history. **Tiller** was Helm 2's server component and has been removed. · 📖 [L13 §11](13-Kubernetes-Troubleshooting-and-Helm.md#11-helm-key-terms)

### Q6. Does Helm scale?
During a traffic spike, which component adds more Pods for an app that was deployed with Helm?
- A) Helm, because it watches load and re-runs `helm upgrade`
- B) The Horizontal Pod Autoscaler (HPA). Helm can only include an HPA in the chart (e.g. `autoscaling.enabled: true`).
- C) The Helm release controller
- D) Nothing: Helm-managed apps can't scale

> ✅ **Answer: B.** Helm packages and deploys. It does nothing while the app runs. Scaling at run time is the HPA's job. · 📖 [L13 §10](13-Kubernetes-Troubleshooting-and-Helm.md#10-helm-why-a-package-manager), [L12 §10](12-Kubernetes-Volumes-and-Autoscaling.md#10-horizontal-pod-autoscaler-hpa)

---

## Part B: Values & Templates

### Q7. Which value wins?
The chart's `values.yaml` has `replicaCount: 1`, `prod.yaml` has `replicaCount: 3`, and you run `helm install shop ./chart -f prod.yaml --set replicaCount=5`. How many replicas?
- A) 1
- B) 3
- C) 5
- D) The install fails because of the conflict

> ✅ **Answer: C.** Precedence, lowest to highest: chart `values.yaml` → `-f` files (later files win) → `--set`. So `--set replicaCount=5` wins. · 📖 [L13 §12](13-Kubernetes-Troubleshooting-and-Helm.md#12-helm-basic-commands)

### Q8. What `.Release.Name` becomes
A template contains `name: {{ .Release.Name }}-web`. You run `helm install demo-release ./demo-chart`. What is the Deployment called?
- A) `demo-chart-web`
- B) `demo-release-web`
- C) `{{ .Release.Name }}-web`
- D) `web`

> ✅ **Answer: B.** `.Release.Name` is the name you give at install time (`demo-release`). `.Chart.Name` would give `demo-chart`. · 📖 [L13 §14](13-Kubernetes-Troubleshooting-and-Helm.md#14-templates--values-how-helm-works)

### Q9. `helm template`
What does `helm template shop ./chart -f prod.yaml` do?
- A) Installs the chart and prints the manifests
- B) Renders the chart's YAML locally and prints it, without creating a release or changing the cluster
- C) Creates a new chart from a template
- D) Validates the YAML against the cluster and saves a draft release

> ✅ **Answer: B.** `helm template` only renders, and doesn't need the cluster. `helm install --dry-run` is different: it talks to the cluster to validate. · 📖 [L13 §14](13-Kubernetes-Troubleshooting-and-Helm.md#14-templates--values-how-helm-works)

### Q10. One chart, several environments
You need the same app in dev (1 replica, `latest` tag) and prod (5 replicas, pinned tag, Ingress on). What's the recommended Helm approach?
- A) Copy the chart into `chart-dev/` and `chart-prod/` and edit each
- B) One chart, plus `values-dev.yaml` and `values-prod.yaml` passed with `-f`
- C) Put `if env == "prod"` checks in every template
- D) Use a long list of `--set` flags typed by hand each time

> ✅ **Answer: B.** Templates stay identical, and only the values change per environment. Values files live in Git, so they're reviewed and repeatable, e.g. `helm upgrade --install shop-prod ./chart -f values-prod.yaml -n prod`. · 📖 [L13 §15](13-Kubernetes-Troubleshooting-and-Helm.md#15-upgrade-rollback--environments)

### Q11. `helm lint`
When is `helm lint ./demo-chart` most useful?
- A) After install, to see which Pods are failing
- B) Before install, to catch mistakes in the chart's structure and templates
- C) To compare two releases
- D) To delete unused values

> ✅ **Answer: B.** `lint` checks the chart itself (`Chart.yaml`, template syntax, values) without installing anything. A good order before deploying is `lint` → `template` → `install`. · 📖 [L13 §14](13-Kubernetes-Troubleshooting-and-Helm.md#14-templates--values-how-helm-works)

---

## Part C: Release Lifecycle

### Q12. Upgrading with new values
Revision 2 of a release was created with `helm upgrade shop ./chart --set replicaCount=3`. Then you run `helm upgrade shop ./chart --set image.tag=1.28`. The chart's default `replicaCount` is 1. How many replicas now?
- A) 3, because Helm merges new `--set` values with the previous ones
- B) 1, because new values replace the old ones and `replicaCount` goes back to the chart default
- C) The upgrade fails until you pass `replicaCount` again
- D) 3, but only until the next rollback

> ✅ **Answer: B.** When you pass new values, Helm builds them from the chart defaults plus **only** what you passed this time. Earlier `--set` values aren't merged. Keep settings in a values file, or use `--reuse-values`. · 📖 [L14 §7](14-Kubernetes-and-Helm-Revision.md#7-helm-commands)

### Q13. Upgrading with no values
Same release: revision 2 had `--set replicaCount=3`. This time you run a plain `helm upgrade shop ./chart`, with **no** `-f` or `--set` at all. How many replicas?
- A) 1, the chart default
- B) 3, because with no new values Helm reuses the previous release's values
- C) 0, because the upgrade clears all values
- D) The upgrade fails without values

> ✅ **Answer: B.** This is the special case that goes with Q12: if you pass **no** values, Helm copies the last release's values. As soon as you pass **any** value, the old ones are no longer carried over. · 📖 [L14 §7](14-Kubernetes-and-Helm-Revision.md#7-helm-commands)

### Q14. Rollback revision number
`helm history shop` shows revisions 1, 2 and 3 (3 is current). You run `helm rollback shop 1`. What does `helm history` show as the newest revision?
- A) Revision 1, marked deployed again
- B) Revision 4, with the same configuration as revision 1
- C) Revision 2
- D) Revision 3, overwritten in place

> ✅ **Answer: B.** Helm history only grows. A rollback is recorded as a **new** revision (4) holding revision 1's configuration. · 📖 [L13 §15](13-Kubernetes-Troubleshooting-and-Helm.md#15-upgrade-rollback--environments)

### Q15. Uninstalling a database chart
You installed a PostgreSQL chart, which runs a StatefulSet with `volumeClaimTemplates`, then ran `helm uninstall pg`. What happens to the database's PVCs?
- A) They're deleted along with everything else in the release
- B) They're kept, because the StatefulSet created them (not Helm), so they aren't part of the release's manifests
- C) They're moved to the `default` namespace
- D) They're deleted only if the chart's `appVersion` changed

> ✅ **Answer: B.** `helm uninstall` removes every object **Helm itself created**. PVCs made from `volumeClaimTemplates` are created by the StatefulSet controller, so they stay, and the data is safe. Delete them yourself if you really want the data gone. ➕ That's also why `helm upgrade --install` (install if missing, else upgrade) is safe to re-run in CI: your data survives. · 📖 [L14 §2](14-Kubernetes-and-Helm-Revision.md#2-statefulsets-new), [L13 §12](13-Kubernetes-Troubleshooting-and-Helm.md#12-helm-basic-commands)
