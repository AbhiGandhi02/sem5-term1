# Course Recap & CI/CD with GitHub Actions

> **DevOps – Lecture 15 notes.** Anything marked **➕ Extra** was not covered in class. It's there because it fills a gap or comes up often in labs and interviews.
>
> This class did two things: (1) a **recap of the whole course**, which is a one-page map in §1 linking every lecture, and (2) a **new topic, CI/CD with GitHub Actions**, which is the rest of this file. CI/CD ties everything together: [Git](04-Git-Version-Control.md) triggers it, [shell scripts](02-Shell-Scripting.md) run in it, [Docker](05-Docker.md) images come out of it, and [Kubernetes](08-Kubernetes-Basics.md) runs them.
>
> **Practice:** [CI/CD MCQs](CICD-MCQs.md).

---

## Table of Contents
1. [Course Recap: The Whole Journey on One Page](#1-course-recap-the-whole-journey-on-one-page)
2. [What Is CI/CD?](#2-what-is-cicd)
3. [GitHub Actions: Key Terms](#3-github-actions-key-terms)
4. [Anatomy of a Workflow File](#4-anatomy-of-a-workflow-file)
5. [Triggers (`on:`)](#5-triggers-on)
6. [Hands-on: The Python Calculator Project](#6-hands-on-the-python-calculator-project)
7. [The CI Workflow (corrected class example)](#7-the-ci-workflow-corrected-class-example)
8. [Artifacts](#8-artifacts)
9. [Secrets](#9-secrets)
10. [➕ Extending to CD: Build & Push a Docker Image](#10--extending-to-cd-build--push-a-docker-image)
11. [➕ Debugging a Failed Workflow](#11--debugging-a-failed-workflow)
12. [Quick Revision Cheat Sheet](#12-quick-revision-cheat-sheet)
13. [Likely Exam / Interview Questions](#13-likely-exam--interview-questions)

---

## 1. Course Recap: The Whole Journey on One Page

```
 Linux & Shell ──▶ Networking ──▶ Git/GitHub ──▶ Docker ──▶ Kubernetes ──▶ Helm ──▶ CI/CD
  (the OS)        (how it talks)  (the code)    (package)   (run at scale)  (package K8s) (automate it all)
```

| # | Topic | Key ideas | Notes |
|---|-------|-----------|-------|
| 1 | **Linux** | The OS almost every server runs. Permissions, users, processes, core commands | [L1](01-Linux-Permissions-and-Commands.md) |
| 2 | **Shell scripting** | Automate repetitive Linux tasks: variables, loops, conditions, functions | [L2](02-Shell-Scripting.md) |
| 3 | **Networking** | IP, subnets/CIDR, ports, DNS, TCP/UDP, HTTP(S), debugging tools | [L3](03-Networking.md) · [revision](03b-Networking-Revision.md) |
| 4 | **Git & GitHub** | Version control, branches, merge/rebase, PRs, collaboration | [L4](04-Git-Version-Control.md) |
| 5–7 | **Docker** | Containers = code + runtime + libraries, isolated. **Objects: image, container, network, volume.** | [L5](05-Docker.md) · [L6](06-Docker-Lifecycle-and-Multi-Stage.md) · [L7](07-Docker-Networking-and-Volumes.md) |
| 8 | **K8s architecture** | **Control plane**: API server, **etcd** (consistent key-value store of all cluster data), scheduler, controller manager. **Workers**: kubelet, kube-proxy, runtime | [L8](08-Kubernetes-Basics.md) |
| 8 | **K8s tools** | **minikube** = a local single-node cluster · **kubectl** = the CLI | [L8 §10](08-Kubernetes-Basics.md#10-hands-on-your-first-pod) |
| 9–10 | **K8s objects** | **Pod** (smallest unit) · **ReplicaSet** (N copies) · **Deployment** (rolling updates) · **Service** (stable access) · DaemonSet · strategies | [L9](09-Kubernetes-Objects-Services-Deployments.md) · [L10](10-Kubernetes-Deployment-Strategies.md) |
| 11 | **Config & routing** | **ConfigMap** / **Secret** (config outside code) · **Ingress** (external HTTP access, one entry point) | [L11](11-Kubernetes-ConfigMaps-Secrets-Ingress.md) |
| 12 | **Storage & scaling** | **PV / PVC** (durable storage) · StorageClass · HPA | [L12](12-Kubernetes-Volumes-and-Autoscaling.md) |
| 13–14 | **Ops & packaging** | Troubleshooting (get → describe → logs → exec) · **StatefulSet** · **Helm** | [L13](13-Kubernetes-Troubleshooting-and-Helm.md) · [L14](14-Kubernetes-and-Helm-Revision.md) |
| 15 | **CI/CD** | Automate build → test → deploy on every push | this file |
| 16 | **End-to-end project** | One Flask app: manual → Docker → K8s → DevSecOps pipeline (pytest, CodeQL, pip-audit, Trivy) | [L16](16-End-to-End-Deployment-Project.md) |
| 17 | **Terraform (IaC)** | Infrastructure as code: providers, resources, variables, outputs, state · init → plan → apply → destroy | [L17](17-Terraform-IaC.md) |
| 18 | **AWS & VPC** | IaaS/PaaS/SaaS · EC2, S3, DynamoDB, IAM · VPC: subnets, route tables, IGW, NAT, SG vs NACL (built with Terraform) | [L18](18-AWS-VPC-and-Core-Services.md) |
| 19 | **Monitoring & GitOps** | Prometheus (pull, PromQL) + Grafana dashboards · GitOps: Git as source of truth, Argo CD sync / prune / self-heal | [L19](19-Monitoring-Observability-and-GitOps.md) |

**Practice:** [Kubernetes MCQs](Kubernetes-MCQs.md).

---

## 2. What Is CI/CD?

**The problem without it:** a developer pushes code. Someone manually runs the tests (or forgets to), builds the app on their laptop ("works on my machine"), copies it to a server and restarts it. It's slow, error-prone and different every time.

**CI/CD** = a **pipeline** that runs **automatically on every change**:

```
 git push ──▶ ┌─────────────── CI ───────────────┐ ┌──────────────── CD ────────────────┐
              │ checkout → install → lint → test │ │ build image → push → deploy staging │──▶ (approve?) ──▶ production
              │       → build → save artifact    │ │                                     │
              └──────────────────────────────────┘ └─────────────────────────────────────┘
                 ❌ any step fails → pipeline stops, developer is notified, nothing broken ships
```

| Term | Meaning |
|------|---------|
| **Continuous Integration (CI)** | Every push/PR is **automatically built and tested**, so problems are caught within minutes, while the change is small |
| **Continuous Delivery** | Every successful build is **ready to release**, but going to **production needs a manual approval** (a button) |
| **Continuous Deployment** | Every successful build goes to **production automatically**, with no human step |

> ⚠️ **Precision point:** the class notes define CD as "automatically deploys all successful builds to testing, staging, or production". That's **Continuous Deployment**. "CD" can also mean **Continuous Delivery** (automatic up to staging, manual to prod). Interviewers like this distinction.
>
> 💡 **Simple words mein:** CI = har baar code daalte hi automatic exam (test) ho jaata hai. Delivery = result ready hai, principal (manager) sign karega tab publish. Deployment = pass hote hi seedha publish.

**Benefits:** faster releases, bugs caught early, every build done the same way, a full history of what was deployed and when.

**Popular tools:** **GitHub Actions** (built into GitHub), GitLab CI, Jenkins, CircleCI, Azure Pipelines, ➕ Argo CD (GitOps for Kubernetes).

---

## 3. GitHub Actions: Key Terms

```
 Repository
 └── .github/workflows/ci.yml            ← WORKFLOW (one YAML file = one automated process)
       on: push                          ← EVENT / TRIGGER (what starts it)
       jobs:
         build:                          ← JOB (runs on its own fresh RUNNER machine)
           runs-on: ubuntu-latest        ← RUNNER (GitHub-hosted VM: Ubuntu / Windows / macOS)
           steps:                        ← STEPS (run in order, share the same machine)
             - uses: actions/checkout@v4     ← ACTION (a reusable, pre-built step)
             - run: ./build.sh               ← SHELL COMMAND (your own step)
```

| Term | Meaning |
|------|---------|
| **Workflow** | An automated process, defined in `.github/workflows/<name>.yml` |
| **Event / trigger** | What starts it: `push`, `pull_request`, a schedule, a manual button … |
| **Job** | A group of steps that runs on **one runner**. Jobs run **in parallel** by default. |
| **Step** | A single task: either `uses:` an action or `run:` a shell command |
| **Runner** | The machine that executes a job. GitHub-hosted = a **fresh VM every time**. ➕ Self-hosted = your own server. |
| **Action** | A reusable building block from the **Marketplace**, e.g. `actions/checkout`, `actions/setup-python` |
| **Artifact** | Files saved from a run for download or for later jobs ([§8](#8-artifacts)) |
| **Secret** | An encrypted value (password, token) used in workflows ([§9](#9-secrets)) |

> ⚠️ **Each job starts on a clean machine.** Files from job A **don't exist** in job B unless you pass them as **artifacts**. This is the most common beginner surprise.

---

## 4. Anatomy of a Workflow File

```yaml
name: CI                          # shown in the Actions tab

on:                               # WHEN to run (§5)
  push:
    branches: [main]

env:                              # ➕ environment variables for all jobs
  PYTHON_VERSION: "3.12"

jobs:
  test:                           # job id (your choice)
    runs-on: ubuntu-latest        # WHERE to run
    steps:                        # WHAT to do, top to bottom
      - name: Get the code        # optional label shown in the logs
        uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:                     # inputs for the action
          python-version: ${{ env.PYTHON_VERSION }}

      - name: Run a script
        run: |                    # "|" = several shell lines
          echo "Branch: ${{ github.ref_name }}"
          python --version

  deploy:
    needs: test                   # ➕ run AFTER "test" succeeds (otherwise jobs run in parallel)
    runs-on: ubuntu-latest
    steps:
      - run: echo "deploying..."
```

| Syntax | Meaning |
|--------|---------|
| `uses: owner/repo@version` | Run a published action (**pin a version**: `@v4`) |
| `with:` | Pass inputs to an action |
| `run:` | Run shell commands (bash on Ubuntu runners) |
| `${{ ... }}` | **Expression**: read contexts like `github.*`, `secrets.*`, `env.*`, `steps.*` |
| `needs:` ➕ | Job dependency → sequential jobs |
| `if:` ➕ | Run a step/job only when a condition is true |

---

## 5. Triggers (`on:`)

```yaml
on: [push]                       # class example: any push to any branch

on:                              # more realistic
  push:
    branches: [main]             # pushes to main
  pull_request:
    branches: [main]             # PRs targeting main (CI checks show up on the PR ✅)
  workflow_dispatch:             # ➕ adds a "Run workflow" button in the Actions tab
  schedule:
    - cron: "0 2 * * *"          # ➕ every day at 02:00 UTC (nightly builds, cleanups)
```

| Trigger | Typical use |
|---------|-------------|
| `push` | Build/test every commit. Deploy when it's `main`. |
| `pull_request` | **Gate merges**: block the PR until tests pass (with branch protection, [L4 §13](04-Git-Version-Control.md#13-team-workflow-feature-branches--pull-requests)) |
| `workflow_dispatch` ➕ | Manual runs, e.g. a "deploy to prod" button |
| `schedule` ➕ | Nightly tests, security scans |
| `push: tags: ['v*']` ➕ | Release when you push a tag like `v1.2.0` |

---

## 6. Hands-on: The Python Calculator Project

**Repository layout:**
```
calculator/
├── app/
│   ├── __init__.py              # empty: makes "app" importable
│   └── calculator.py
├── tests/
│   └── test_calculator.py
├── requirements.txt
├── build.sh
├── Dockerfile                   # ➕ used in §10
└── .github/
    └── workflows/
        └── ci.yml
```

**`app/calculator.py`**
```python
def add(a, b):
    return a + b

def subtract(a, b):
    return a - b

def multiply(a, b):
    return a * b

def divide(a, b):
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return a / b

if __name__ == "__main__":
    print("2 + 3 =", add(2, 3))
    print("7 - 4 =", subtract(7, 4))
    print("6 * 5 =", multiply(6, 5))
    print("8 / 2 =", divide(8, 2))
```

**`tests/test_calculator.py`** (➕ CI is only useful if there are tests to run)
```python
import pytest
from app.calculator import add, subtract, multiply, divide

def test_add():
    assert add(2, 3) == 5

def test_subtract():
    assert subtract(7, 4) == 3

def test_multiply():
    assert multiply(6, 5) == 30

def test_divide():
    assert divide(8, 2) == 4

def test_divide_by_zero():
    with pytest.raises(ValueError):
        divide(1, 0)
```

**`requirements.txt`**
```
pytest
```

**Run locally (from class):**
```bash
pip install -r requirements.txt
python3 app/calculator.py          # 2 + 3 = 5 ...
python3 -m pytest                  # ➕ 5 passed
```

**`build.sh`**: the same steps CI will run (tie-in to [L2 – Shell Scripting](02-Shell-Scripting.md))
```bash
#!/bin/bash
set -e                                    # stop at the first failing command → the CI step fails

echo "Installing dependencies..."
pip install -r requirements.txt

echo "Running tests..."
mkdir -p build
python3 -m pytest --junitxml=build/test-report.xml

echo "Packaging..."
cp -r app build/
echo "Built from commit ${GITHUB_SHA:-local} on $(date)" > build/build-info.txt
echo "Build complete ✅"
```

> **Why `set -e` matters:** GitHub marks a step **failed** if its command exits non-zero. Without `set -e`, a failing `pytest` in the middle of the script would be ignored if a later line succeeds, and broken code would pass CI.

---

## 7. The CI Workflow (corrected class example)

**`.github/workflows/ci.yml`**
```yaml
name: Hello GitHub Actions
on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4            # 1. get the repo's code onto the runner

      - uses: actions/setup-python@v5        # ➕ 2. a known Python version
        with:
          python-version: "3.12"

      - name: Run build                      # 3. our script: install, test, package
        run: |
          chmod +x build.sh
          ./build.sh

      - name: Upload artifact                # 4. save build/ for download
        if: always()                         # ➕ upload the test report even if tests failed
        uses: actions/upload-artifact@v4
        with:
          name: build-result
          path: build/
```

> ⚠️ **Correction to the class YAML:** it used `actions/checkout@v2` and `actions/upload-artifact@v2`. **`upload-artifact` v1–v3 have been shut down by GitHub**, so a workflow using them now **fails**. `checkout@v2` runs on an old, deprecated Node version. Use **`checkout@v4`** and **`upload-artifact@v4`** (plus `download-artifact@v4`).

**What happens on `git push`:**
```
 1. GitHub sees the push event → finds .github/workflows/ci.yml
 2. Starts a fresh ubuntu-latest VM (the runner)
 3. checkout → setup-python → ./build.sh → upload-artifact
 4. Actions tab: ✅ green tick (or ❌ red cross) next to the commit, also shown on PRs
```

**`chmod +x build.sh`**: the runner checks out your files, and the script needs execute permission ([L1 – permissions](01-Linux-Permissions-and-Commands.md)). ➕ You can save that step permanently with `git update-index --chmod=+x build.sh` and a commit.

---

## 8. Artifacts

**Artifacts** are files a workflow run **saves** after it finishes: build output, test reports, logs, coverage files. Remember, the runner VM is **destroyed** after the job.

```yaml
- uses: actions/upload-artifact@v4
  with:
    name: build-result          # name shown in the run summary
    path: build/                # file(s) or folder(s) to save
    retention-days: 7           # ➕ default 90 days (set by repo/org)
```
- **Download:** Actions tab → open the run → **Artifacts** section at the bottom → `build-result.zip`.
- **➕ Pass files between jobs** (each job has a fresh machine):
```yaml
  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: build-result
          path: build/
      - run: ls build/          # test-report.xml, app/, build-info.txt
```

| ➕ Artifact | Cache (`actions/cache`) |
|------------|-------------------------|
| **Output** you want to keep or download (reports, binaries) | **Speed-up**: reuse downloaded dependencies (pip, npm) between runs |
| Visible in the run summary | Invisible, purely for performance |

---

## 9. Secrets

**Secrets** store sensitive values (API tokens, passwords, cloud keys) **encrypted** in GitHub, so they **never appear in your code**.

**Create one:** repo → **Settings → Secrets and variables → Actions → New repository secret** → name `DOCKERHUB_TOKEN`, value `…`.

**Use it in a workflow:**
```yaml
- name: Log in to Docker Hub
  run: echo "${{ secrets.DOCKERHUB_TOKEN }}" | docker login -u "${{ secrets.DOCKERHUB_USERNAME }}" --password-stdin
```
(The class notes write this as `secrets.YOUR_SECRET_NAME`, always inside `${{ }}`.)

| Fact | Detail |
|------|--------|
| **Masked in logs** | If a secret's value appears in output, GitHub prints `***` |
| **Write-only** | After saving, nobody (not even admins) can view the value, only replace it |
| **Not passed to forks** ➕ | Workflows triggered by PRs from **forked** repos don't get secrets, so strangers can't steal them |
| **`GITHUB_TOKEN`** ➕ | An automatic, short-lived token for the repo itself (comment on PRs, push to GHCR). No setup needed. |
| **Variables** ➕ | Non-secret config (`vars.APP_ENV`), like a ConfigMap vs. a Secret ([L11 §7](11-Kubernetes-ConfigMaps-Secrets-Ingress.md#7-configmap-vs-secret)) |
| **Environments** ➕ | Group secrets per environment (`staging`, `production`) and add **required reviewers**. That's how Continuous **Delivery**'s manual approval is done. |

> ⚠️ **Never** hard-code tokens in the YAML, `echo` a secret on purpose, or commit `.env` files ([L4 §12](04-Git-Version-Control.md#12-best-practices--gitignore)). Use **least privilege**: a Docker Hub **access token**, not your account password.

---

## 10. ➕ Extending to CD: Build & Push a Docker Image

The natural next step: after the tests pass on `main`, **package the app as a Docker image** ([L5](05-Docker.md)) and push it to Docker Hub.

**`Dockerfile`**
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY app/ ./app/
CMD ["python", "app/calculator.py"]
```

**Add a second job to `ci.yml`:**
```yaml
  docker:
    needs: build                                    # only if tests passed
    if: github.ref == 'refs/heads/main'             # only for main, not feature branches
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - uses: docker/build-push-action@v6
        with:
          context: .
          push: true
          tags: |
            ${{ secrets.DOCKERHUB_USERNAME }}/calculator:latest
            ${{ secrets.DOCKERHUB_USERNAME }}/calculator:${{ github.sha }}
```

**Tagging with `github.sha`** (the commit ID) means every image is traceable to the exact code that built it, and you can roll back to any commit's image. That's better than relying on `latest` ([L6 §4](06-Docker-Lifecycle-and-Multi-Stage.md#4-working-with-images-pull-images-rmi)).

**The full picture: from push to Kubernetes**
```
 git push main ──▶ CI: test ✅ ──▶ build image calculator:<sha> ──▶ push to registry
                                                                         │
                         ┌───────────────────────────────────────────────┘
                         ▼
     deploy step: kubectl set image deploy/calculator app=user/calculator:<sha>
              or: helm upgrade --install calculator ./chart --set image.tag=<sha>
                         │
                         ▼
     Kubernetes rolling update (see L10) → zero downtime ✅
```
> ➕ In industry, the last step is often done with **GitOps** (Argo CD / Flux): CI updates the image tag in a Git repo, and a tool inside the cluster syncs the cluster to match Git.

---

## 11. ➕ Debugging a Failed Workflow

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Workflow **doesn't run at all** | File not in `.github/workflows/`, wrong extension, YAML indentation error, branch filter doesn't match | Check the path, validate the YAML, check `on:` |
| `Permission denied: ./build.sh` | Script not executable | `chmod +x build.sh` step, or commit with `git update-index --chmod=+x` |
| `This request has been automatically failed because it uses a deprecated version of actions/upload-artifact` | Old `@v2`/`@v3` action | Upgrade to `@v4` |
| `ModuleNotFoundError: No module named 'app'` | Tests run from the wrong folder / no `__init__.py` | Run `python -m pytest` from the repo root, add `app/__init__.py` |
| Docker login fails / secret empty | Secret name typo, or a PR from a fork | Check **Settings → Secrets**. Names are case-sensitive. |
| Step "passes" though tests failed | Script doesn't exit non-zero | Add `set -e` to shell scripts |

**Where to look:** Actions tab → the failed run → the failed **job** → expand the failed **step**. The red line shows the exact error. ➕ "Re-run jobs" → "Enable debug logging" gives more detail.

---

## 12. Quick Revision Cheat Sheet

```
COURSE        Linux → Shell → Networking → Git → Docker (image/container/network/volume)
              → K8s (control plane: API, etcd, scheduler, CM · workers: kubelet, kube-proxy)
              → objects (Pod, RS, Deploy, StatefulSet, Service, ConfigMap, Secret, Ingress, PV/PVC)
              → Helm → CI/CD · minikube = local cluster · kubectl = CLI
CI            every push → build + test automatically → catch bugs early
CD            Delivery = ready to release, MANUAL approval to prod · Deployment = AUTO to prod
GHA TERMS     workflow (.github/workflows/*.yml) · event (on:) · job (own runner, parallel)
              step (uses: action | run: shell) · runner (fresh VM) · action (marketplace) · artifact · secret
TRIGGERS      push · pull_request · workflow_dispatch (button) · schedule (cron) · tags
SYNTAX        runs-on · steps · uses: x@v4 · with: · run: | · ${{ expr }} · needs: · if:
CLASS FIX     checkout@v2 → @v4 · upload-artifact@v2 → @v4 (v1–v3 shut down) · setup-python@v5
BUILD.SH      set -e (fail fast) · chmod +x · pip install -r · pytest --junitxml · cp to build/
ARTIFACTS     upload-artifact (path, name, retention) · download from run page / download-artifact in next job
              jobs DON'T share files otherwise · artifact = output, cache = speed
SECRETS       Settings → Secrets and variables → Actions · ${{ secrets.NAME }} · masked *** · write-only
              no secrets for fork PRs · GITHUB_TOKEN automatic · vars.* for non-secret · environments + reviewers
CD ➕         needs: build · if: main · docker/login-action · build-push-action · tag :${{ github.sha }}
              → kubectl set image / helm upgrade → rolling update · GitOps = Argo CD / Flux
```

---

## 13. Likely Exam / Interview Questions

1. **What is CI/CD, and why is it important?** An automated pipeline that builds, tests and deploys every change. It catches bugs early and makes releases fast, repeatable and consistent.
2. **Continuous Integration vs. Continuous Delivery vs. Continuous Deployment?** Automatic build and test / always releasable, with manual approval for production / automatic release to production.
3. **Where do GitHub Actions workflows live?** In `.github/workflows/*.yml` in the repository.
4. **Define workflow, job, step, runner and action.** File-level process / a set of steps on one runner / a single task / the VM that runs a job / a reusable published step.
5. **Do jobs run in sequence or in parallel? How do you make one wait for another?** In parallel by default. Use `needs: <job>`.
6. **Job B can't find files built in job A. Why, and what's the fix?** Each job runs on a fresh runner. Upload them as an artifact in A and download it in B.
7. **What does `actions/checkout` do, and why is it usually the first step?** It clones the repository onto the runner, which starts empty.
8. **`uses:` vs. `run:`?** `uses:` runs a pre-built action. `run:` runs your own shell commands.
9. **What are artifacts? Give examples.** Files saved from a run for download or for later jobs: build output, test reports, logs.
10. **How do you use a password in a workflow without putting it in the code?** Store it as a repository secret and reference it with `${{ secrets.NAME }}`. It's encrypted and masked in logs.
11. **Why aren't secrets available to pull requests from forks?** To stop untrusted code from reading and leaking them.
12. **The class workflow uses `upload-artifact@v2` and now fails. Why?** GitHub has shut down v1–v3 of the artifact actions. Upgrade to `@v4`.
13. **Why put `set -e` in a CI shell script?** So the script, and therefore the step, fails on the first failing command instead of hiding test failures.
14. **How would you trigger a workflow manually, or every night?** `workflow_dispatch` and `schedule` with a cron expression.
15. **Why tag Docker images with the commit SHA instead of only `latest`?** It's traceable to the exact code, the deploys are reproducible, and rollback is easy.
16. **Describe a full pipeline from `git push` to Kubernetes.** Push → test → build image → push to registry → `kubectl set image` or `helm upgrade` → rolling update.
17. **Name three components of the Kubernetes control plane and what etcd stores.** API server, scheduler, controller manager. etcd is a consistent, highly available key-value store of all cluster data.
18. **What are minikube and kubectl?** A local single-node Kubernetes cluster, and the command-line tool for talking to any cluster.
