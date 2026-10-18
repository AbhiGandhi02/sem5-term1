# End-to-End Project: Deploying a Python Dashboard (Manual → Docker → Kubernetes → DevSecOps CI/CD)

> **DevOps – Lecture 16 notes.** Anything marked **➕ Extra** was not covered in class. It's there because it fills a gap or comes up often in labs and interviews.
>
> **This lecture uses the whole course on one app.** Each stage links back to its lecture: [Docker](05-Docker.md) · [Kubernetes](09-Kubernetes-Objects-Services-Deployments.md) · [CI/CD](15-CICD-with-GitHub-Actions.md) · [image security](06-Docker-Lifecycle-and-Multi-Stage.md#9-docker--devsecops-secure-images).
>
> **Next:** [Lecture 17 – Terraform (IaC)](17-Terraform-IaC.md).

---

## Table of Contents
1. [The Big Picture: Four Ways to Run the Same App](#1-the-big-picture-four-ways-to-run-the-same-app)
2. [The Application](#2-the-application)
3. [Stage 1: Run It Manually](#3-stage-1-run-it-manually)
4. [Stage 2: Run It in Docker](#4-stage-2-run-it-in-docker)
5. [Stage 3: Run It on Kubernetes (minikube)](#5-stage-3-run-it-on-kubernetes-minikube)
6. [Stage 4: The DevSecOps CI/CD Pipeline](#6-stage-4-the-devsecops-cicd-pipeline)
7. [Pipeline Stage: Unit Tests & 100% Coverage](#7-pipeline-stage-unit-tests--100-coverage)
8. [Pipeline Stage: SAST with CodeQL](#8-pipeline-stage-sast-with-codeql)
9. [Pipeline Stage: SCA with pip-audit](#9-pipeline-stage-sca-with-pip-audit)
10. [Pipeline Stage: Build → Trivy Scan → Push](#10-pipeline-stage-build--trivy-scan--push)
11. [The Complete Workflow File](#11-the-complete-workflow-file)
12. [Troubleshooting & Debugging](#12-troubleshooting--debugging)
13. [Quick Revision Cheat Sheet](#13-quick-revision-cheat-sheet)
14. [Likely Exam / Interview Questions](#14-likely-exam--interview-questions)

---

## 1. The Big Picture: Four Ways to Run the Same App

```
 Stage 1: Manual        python app.py                      "works on my machine"
     │
 Stage 2: Docker        docker run -p 5001:5001 ...         works on ANY machine with Docker
     │
 Stage 3: Kubernetes    kubectl apply -f deployment.yaml    replicas, self-healing, a Service
     │
 Stage 4: CI/CD         git push → test → scan → build → scan → push    automatic AND secure
```

| Stage | What you gain | What's still manual |
|-------|---------------|---------------------|
| Manual | Fast to try | Setup on every machine, nothing repeatable |
| Docker | Same environment everywhere | You build, run and update by hand |
| Kubernetes | Scaling, self-healing, rolling updates | Building and pushing images |
| **CI/CD** | **Every push is tested, scanned, built and published automatically** | Nothing (except approving prod, if you choose) |

> 💡 **Simple words mein:** pehle khud khana banaya (manual), phir tiffin mein pack kiya (Docker), phir restaurant khola jo kitne bhi customers sambhale (Kubernetes), aur end mein factory line lagayi jo har order ko check karke automatic pack kare (CI/CD).

---

## 2. The Application

A small **Flask** app: it serves a **dashboard frontend** (static HTML/CSS/JS) and **JSON APIs**.

```
python-dashboard/
├── app.py                    # entry point: Flask app + all routes
├── requirements.txt
├── static/
│   ├── index.html            # dashboard page
│   ├── style.css
│   └── script.js             # calls the APIs with fetch()
├── tests/
│   └── test_app.py
├── Dockerfile
├── .dockerignore             # ➕
├── k8s/
│   ├── deployment.yaml
│   └── service.yaml
└── .github/workflows/
    └── pipeline.yml
```

### `app.py`
```python
from flask import Flask, jsonify, request, send_from_directory

app = Flask(__name__, static_folder="static")


@app.route("/")
def index():
    return send_from_directory(app.static_folder, "index.html")   # the dashboard


@app.route("/api/health")
def health():
    return jsonify(status="healthy")                              # used by K8s probes


@app.route("/api/greet/<name>")
def greet(name):
    return jsonify(message=f"Hello, {name}! Welcome to our dashboard!")


@app.route("/api/calculate", methods=["POST"])
def calculate():
    data = request.get_json(silent=True) or {}
    a, b, op = data.get("a"), data.get("b"), data.get("op")
    if not isinstance(a, (int, float)) or not isinstance(b, (int, float)):
        return jsonify(error="a and b must be numbers"), 400
    if op == "add":
        result = a + b
    elif op == "subtract":
        result = a - b
    elif op == "multiply":
        result = a * b
    elif op == "divide":
        if b == 0:
            return jsonify(error="Cannot divide by zero"), 400
        result = a / b
    else:
        return jsonify(error=f"Unknown operation: {op}"), 400
    return jsonify(result=result)


@app.route("/api/pipeline")
def pipeline():
    stages = ["checkout", "test", "sast", "sca", "build", "scan", "push"]
    return jsonify(stages=[{"name": s, "status": "success"} for s in stages])


if __name__ == "__main__":  # pragma: no cover
    app.run(host="0.0.0.0", port=5001)
```

| Method | Endpoint | Returns |
|--------|----------|---------|
| GET | `/` | Dashboard HTML |
| GET | `/api/health` | `{"status": "healthy"}` |
| GET | `/api/greet/<name>` | Custom greeting |
| POST | `/api/calculate` | `{"a": 6, "b": 3, "op": "divide"}` → `{"result": 2.0}` |
| GET | `/api/pipeline` | Simulated pipeline stages |

> ⚠️ **Two corrections to the class snippet:**
> 1. The snippet's route is `/greet/<name>`, but the endpoint list says `/api/greet/<name>`. Pick one. These notes use `/api/greet/<name>`, so every API sits under `/api/`.
> 2. **`host="0.0.0.0"` is essential.** Flask's default is `127.0.0.1`, which only accepts connections from **inside** the container, so `docker run -p` and Kubernetes would show nothing ([L5 §9](05-Docker.md#9-port-mapping--docker-networking)).
>
> ➕ **Why port 5001, not Flask's usual 5000?** On macOS, port 5000 is taken by **AirPlay Receiver**.

### `static/script.js` (excerpt): how the frontend uses the API
```js
async function greet() {
  const name = document.getElementById("name").value;
  const res = await fetch(`/api/greet/${encodeURIComponent(name)}`);
  document.getElementById("output").textContent = (await res.json()).message;
}
```

### `requirements.txt`
```
flask==3.0.3
```
➕ Dev/CI tools are kept separate in `requirements-dev.txt`: `pytest`, `pytest-cov`, `pip-audit`. That keeps the production image small.

---

## 3. Stage 1: Run It Manually

```bash
python3 -m venv venv                  # isolated environment for this project
source venv/bin/activate              # Windows: venv\Scripts\activate
pip install -r requirements.txt
python app.py                         # * Running on http://0.0.0.0:5001
```
```bash
# Test it
open http://localhost:5001                         # dashboard
curl localhost:5001/api/health                     # {"status":"healthy"}
curl localhost:5001/api/greet/Aarham               # {"message":"Hello, Aarham! Welcome..."}
curl -X POST localhost:5001/api/calculate \
     -H "Content-Type: application/json" \
     -d '{"a": 6, "b": 3, "op": "multiply"}'       # {"result":18}
```

> **Virtual environment = per-project Python libraries.** Without it, every project shares one global set of packages, and versions clash.
>
> ⚠️ `python app.py` uses Flask's **development server**. It's fine for learning, not for production. ➕ In production, run it with **gunicorn**: `gunicorn -b 0.0.0.0:5001 app:app`.

---

## 4. Stage 2: Run It in Docker

### `Dockerfile`
```dockerfile
FROM python:3.12-slim                         # small official base image

WORKDIR /app

COPY requirements.txt .                       # dependencies first → layer caching (L5 §6)
RUN pip install --no-cache-dir -r requirements.txt

COPY . .                                      # then the app code

RUN useradd -m appuser                        # ➕ don't run as root (L6 §9)
USER appuser

EXPOSE 5001
CMD ["python", "app.py"]
```

### ➕ `.dockerignore`
```
venv/
__pycache__/
.git/
tests/
.github/
```

### Build, run, check
```bash
docker build -t python-web .
docker run -d -p 5001:5001 --name dashboard python-web     # -d (➕) frees the terminal
curl localhost:5001/api/health                             # {"status":"healthy"} ✅
docker logs dashboard
docker rm -f dashboard
```

> Why `COPY requirements.txt` comes before `COPY . .`: [L5 §6 – layer caching](05-Docker.md#6-layers--caching). Editing `app.py` then rebuilds in seconds, without reinstalling Flask.

---

## 5. Stage 3: Run It on Kubernetes (minikube)

### Getting the image into minikube ➕ (the step everyone misses)
minikube runs its **own** container runtime, so an image you built on your laptop is **invisible** to it. Without this step you get `ErrImagePull` / `ImagePullBackOff` ([L13 §7.2](13-Kubernetes-Troubleshooting-and-Helm.md#72-imagepullbackoff--errimagepull-cant-get-the-image)).
```bash
minikube image load python-web:latest           # copy your local image into minikube
# or: eval $(minikube docker-env) && docker build -t python-web .   (build straight inside minikube)
# or: push to Docker Hub and use <user>/python-web:<tag>            (what the pipeline does)
```

### `k8s/deployment.yaml`
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: python-web
spec:
  replicas: 2
  selector:
    matchLabels:
      app: python-web
  template:
    metadata:
      labels:
        app: python-web
    spec:
      containers:
        - name: python-web
          image: python-web:latest
          imagePullPolicy: IfNotPresent        # ➕ use the loaded image, don't try Docker Hub
          ports:
            - containerPort: 5001
          readinessProbe:                      # ➕ only send traffic when healthy (L10 §2)
            httpGet: { path: /api/health, port: 5001 }
          livenessProbe:                       # ➕ restart if it hangs
            httpGet: { path: /api/health, port: 5001 }
            initialDelaySeconds: 10
          resources:                           # ➕ needed for HPA and good scheduling (L12)
            requests: { cpu: 100m, memory: 128Mi }
            limits:   { cpu: 500m, memory: 256Mi }
```
> ⚠️ With the tag **`:latest`**, Kubernetes defaults to `imagePullPolicy: Always` and tries Docker Hub even though you loaded the image. That's why `IfNotPresent` is set explicitly.

### `k8s/service.yaml`
```yaml
apiVersion: v1
kind: Service
metadata:
  name: python-web-service
spec:
  type: NodePort
  selector:
    app: python-web                # must match the Pod labels
  ports:
    - port: 80                     # Service port
      targetPort: 5001             # container port (where Flask listens)
      nodePort: 30001              # optional; omit and K8s picks one in 30000–32767
```

### Apply & access
```bash
kubectl apply -f k8s/deployment.yaml -f k8s/service.yaml
kubectl get pods -l app=python-web          # 2/2 Running, READY 1/1
kubectl get endpoints python-web-service    # 2 Pod IPs:5001 = the selector works ✅
minikube service python-web-service         # opens the browser at the NodePort URL
# ➕ alternative: kubectl port-forward svc/python-web-service 8080:80 → http://localhost:8080
```
Port chain: **nodePort 30001 → port 80 → targetPort 5001 = containerPort 5001** ([L9 §10](09-Kubernetes-Objects-Services-Deployments.md#10-port-mapping-containerport-targetport-port-nodeport)).

---

## 6. Stage 4: The DevSecOps CI/CD Pipeline

**DevSecOps** = putting **security checks inside** the pipeline instead of at the end. The idea is called **"shift left"**: find problems **early** (left on the timeline), when they're cheapest to fix.

```
 git push
    │
    ├──▶ [test]   pytest + coverage 100%  ──┐
    ├──▶ [sast]   CodeQL (your code)        ├──▶ all pass? ──▶ [docker] build :sha ──▶ Trivy scan ──▶ push to Docker Hub
    └──▶ [sca]    pip-audit (dependencies) ─┘                                              ❌ HIGH/CRITICAL → stop
          (run in parallel)                                   (needs: test, sast, sca)
```

| Check | Looks at | Finds | Tool in class |
|-------|----------|-------|---------------|
| **Unit tests** | Behaviour | Bugs, regressions | **pytest** (+ pytest-cov) |
| **SAST**: Static Application Security Testing | **Your source code** (without running it) | SQL injection, command injection, unsafe code patterns | **CodeQL** |
| **SCA**: Software Composition Analysis | **Third-party dependencies** | Libraries with known CVEs | **pip-audit** |
| **Container scanning** | **The built image** (OS packages + libraries) | CVEs in the base image and system packages | **Trivy** |
| ➕ DAST: Dynamic testing | The **running** app, attacked from outside | Live issues (XSS, misconfiguration) | OWASP ZAP |

> 💡 **Simple words mein:** SAST = apni likhi copy check karna. SCA = jo cheezein bazaar se laaye (libraries) unki expiry check karna. Trivy = poora packed dabba (image) scan karna. Teeno alag galtiyan pakadte hain.

---

## 7. Pipeline Stage: Unit Tests & 100% Coverage

### `tests/test_app.py`
```python
import pytest
from app import app


@pytest.fixture
def client():
    app.config["TESTING"] = True
    return app.test_client()                     # fake HTTP client, no server needed


def test_index(client):
    assert client.get("/").status_code == 200

def test_health(client):
    assert client.get("/api/health").get_json() == {"status": "healthy"}

def test_greet(client):
    assert "Hello, Aarham" in client.get("/api/greet/Aarham").get_json()["message"]

@pytest.mark.parametrize("op, expected", [
    ("add", 9), ("subtract", 3), ("multiply", 18), ("divide", 2),
])
def test_calculate(client, op, expected):
    res = client.post("/api/calculate", json={"a": 6, "b": 3, "op": op})
    assert res.get_json()["result"] == expected

def test_divide_by_zero(client):
    assert client.post("/api/calculate", json={"a": 1, "b": 0, "op": "divide"}).status_code == 400

def test_bad_input(client):
    assert client.post("/api/calculate", json={"a": "x", "b": 1, "op": "add"}).status_code == 400

def test_unknown_op(client):
    assert client.post("/api/calculate", json={"a": 1, "b": 1, "op": "pow"}).status_code == 400

def test_pipeline(client):
    assert len(client.get("/api/pipeline").get_json()["stages"]) == 7
```

### Run with coverage
```bash
pip install pytest pytest-cov
python -m pytest tests/ -v --cov=app --cov-report=term-missing --cov-fail-under=100   # "python -m" puts the repo root on the import path
```
```
Name     Stmts   Miss  Cover   Missing
--------------------------------------
app.py      ..      0   100%
Required test coverage of 100% reached. Total coverage: 100.00%
```
| Flag | Meaning |
|------|---------|
| `--cov=app` | Measure which lines of `app.py` the tests ran |
| `--cov-report=term-missing` | List the **line numbers** never tested |
| `--cov-fail-under=100` | **Fail the pipeline** if coverage drops below 100% |

> **Coverage** = the percentage of code lines executed by tests. `# pragma: no cover` leaves out lines that can't sensibly be tested (the `if __name__ == "__main__"` block).
>
> ➕ 100% coverage doesn't mean bug-free. It only means every line **ran**, not that every case was **checked**. Many teams use 80–90%. The class set 100% as a strict quality gate.

---

## 8. Pipeline Stage: SAST with CodeQL

**CodeQL** (by GitHub) turns your code into a database and runs **security queries** on it, e.g. "user input reaches an SQL query or shell command without sanitising".

```yaml
  sast:
    runs-on: ubuntu-latest
    permissions:
      security-events: write          # ➕ required: lets CodeQL upload its results
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: github/codeql-action/init@v3
        with:
          languages: python
      - uses: github/codeql-action/analyze@v3
```
- Results appear under **Security → Code scanning alerts** in the repo.
- ➕ Free for **public** repos. Private repos need GitHub Advanced Security.

**What it would catch:**
```python
# ❌ command injection: CodeQL flags this
os.system("ping " + request.args["host"])
# ✅ safe
subprocess.run(["ping", "-c", "1", host], check=True)
```

---

## 9. Pipeline Stage: SCA with pip-audit

Your app is maybe 50 lines, but Flask and its dependencies are **thousands of lines you didn't write**. **pip-audit** checks every package version in `requirements.txt` against public vulnerability databases (PyPI advisories, OSV).

```yaml
  sca:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install pip-audit
      - run: pip-audit -r requirements.txt       # exits non-zero → job fails if a known CVE is found
```
```
Found 1 known vulnerability in 1 package
Name   Version  ID                   Fix Versions
-----  -------  -------------------  ------------
flask  2.2.0    PYSEC-2023-62        2.2.5,2.3.2
```
**Fix:** bump the version in `requirements.txt` to a fixed one, then re-run.

> ➕ **Pin versions** (`flask==3.0.3`), so you know exactly what you ship and SCA results are repeatable. **Dependabot** (built into GitHub) can open PRs automatically to bump vulnerable dependencies.

---

## 10. Pipeline Stage: Build → Trivy Scan → Push

**Order matters: scan BEFORE pushing**, so a vulnerable image never reaches the registry.

```yaml
  docker:
    needs: [test, sast, sca]                       # only if ALL checks passed
    if: github.ref == 'refs/heads/main'            # only publish from main
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build image with a unique SHA tag
        run: docker build -t ${{ secrets.DOCKERHUB_USERNAME }}/python-web:${{ github.sha }} .

      - name: Scan image with Trivy
        uses: aquasecurity/trivy-action@0.28.0     # pin a release tag
        with:
          image-ref: ${{ secrets.DOCKERHUB_USERNAME }}/python-web:${{ github.sha }}
          severity: CRITICAL,HIGH
          exit-code: "1"                           # fail the job on HIGH/CRITICAL findings
          ignore-unfixed: true                     # ➕ skip CVEs with no available fix yet

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Push image
        run: docker push ${{ secrets.DOCKERHUB_USERNAME }}/python-web:${{ github.sha }}
```

| Practice | Why |
|----------|-----|
| **Unique SHA tag** (`:${{ github.sha }}`) | Every image maps to one exact commit, so deploys are traceable and rollback is easy ([L15 §10](15-CICD-with-GitHub-Actions.md#10--extending-to-cd-build--push-a-docker-image)) |
| **Trivy before push** | Vulnerable images never get published |
| `exit-code: 1` | Turns findings into a **failed** pipeline, not just a warning |
| Secrets for Docker Hub | No credentials in code ([L15 §9](15-CICD-with-GitHub-Actions.md#9-secrets)) |

**Run Trivy locally too:**
```bash
trivy image --severity HIGH,CRITICAL python-web:latest
```
If it finds OS-package CVEs: use a newer/slimmer base image (`python:3.12-slim` instead of `python:3.12`), rebuild to get patches, or try a multi-stage build ([L6 §8](06-Docker-Lifecycle-and-Multi-Stage.md#8-multi-stage-builds)).

---

## 11. The Complete Workflow File

**`.github/workflows/pipeline.yml`**
```yaml
name: DevSecOps Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v4                # class used @v2: outdated
      - name: Set up Python
        uses: actions/setup-python@v5            # class used @v2: outdated
        with:
          python-version: "3.12"
      - name: Run tests with coverage
        run: |
          pip install -r requirements.txt pytest pytest-cov
          python -m pytest tests/ -v --cov=app --cov-report=term-missing --cov-fail-under=100

  sast:
    runs-on: ubuntu-latest
    permissions:
      security-events: write
      contents: read
    steps:
      - uses: actions/checkout@v4
      - uses: github/codeql-action/init@v3
        with:
          languages: python
      - uses: github/codeql-action/analyze@v3

  sca:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: |
          pip install pip-audit
          pip-audit -r requirements.txt

  docker:
    needs: [test, sast, sca]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    env:
      IMAGE: ${{ secrets.DOCKERHUB_USERNAME }}/python-web:${{ github.sha }}
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t "$IMAGE" .
      - uses: aquasecurity/trivy-action@0.28.0
        with:
          image-ref: ${{ env.IMAGE }}
          severity: CRITICAL,HIGH
          exit-code: "1"
          ignore-unfixed: true
      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - run: docker push "$IMAGE"
```

**Before the first run:** add the `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` repository secrets (Settings → Secrets and variables → Actions).

**➕ The final step (CD):** point the Kubernetes Deployment at the new image:
```bash
kubectl set image deploy/python-web python-web=<user>/python-web:<sha>   # rolling update, zero downtime
```

---

## 12. Troubleshooting & Debugging

### Verbose output (from class)
| Where | How |
|-------|-----|
| **pytest** | `pytest -v` (each test name) · **`pytest -vv`** (full assertion diffs) · ➕ `-x` stop at first failure · `-s` show `print()` output |
| **GitHub Actions** | Re-run the job → **"Enable debug logging"**, or ➕ set repository secret/variable `ACTIONS_STEP_DEBUG` = `true` |

### ➕ Common problems in this project
| Symptom | Cause | Fix |
|---------|-------|-----|
| Docker container runs, browser shows nothing | Flask bound to `127.0.0.1` | `app.run(host="0.0.0.0", port=5001)` |
| `Address already in use :5000` on Mac | AirPlay Receiver | Use 5001 (as in class) |
| K8s Pods `ErrImagePull` with a local image | minikube can't see your laptop's images, or `:latest` forces a pull | `minikube image load` + `imagePullPolicy: IfNotPresent` |
| `minikube service` opens nothing / connection refused | `targetPort` ≠ 5001, or selector mismatch | `kubectl get endpoints python-web-service` |
| Pipeline fails at coverage | A new route has no test | `--cov-report=term-missing` shows the line numbers → add a test |
| `ModuleNotFoundError: No module named 'app'` in CI | pytest can't find `app.py` | Use `python -m pytest` from the repo root (➕ or add an empty `conftest.py` in the root) |
| CodeQL: `Resource not accessible by integration` | Missing permission | `permissions: security-events: write` |
| Trivy fails the build | HIGH/CRITICAL CVEs in the base image | Update the base image, use `-slim`, rebuild |
| Docker push `unauthorized` | Wrong secret name or password instead of token | Check the secrets. Use a Docker Hub **access token**. |

---

## 13. Quick Revision Cheat Sheet

```
APP           Flask app.py · static/ frontend · /api/health /api/greet/<name> /api/calculate (POST) /api/pipeline
              app.run(host="0.0.0.0", port=5001) · 5001 because macOS AirPlay uses 5000
MANUAL        python3 -m venv venv · source venv/bin/activate · pip install -r requirements.txt · python app.py
DOCKER        FROM python:3.12-slim · COPY requirements → pip install → COPY . · EXPOSE 5001 · CMD python app.py
              docker build -t python-web . · docker run -p 5001:5001 python-web
K8S           minikube image load python-web · imagePullPolicy: IfNotPresent (:latest = Always!)
              Deployment (replicas, containerPort 5001, probes /api/health) · Service NodePort → targetPort 5001
              kubectl apply -f deployment.yaml -f service.yaml · minikube service python-web-service
DEVSECOPS     security inside the pipeline · shift left · test ‖ SAST ‖ SCA → build → scan → push
TESTS         pytest + pytest-cov · --cov=app --cov-fail-under=100 · # pragma: no cover · Flask test_client
SAST          CodeQL = scans YOUR code · init@v3 + analyze@v3 · permissions: security-events: write
SCA           pip-audit -r requirements.txt = scans DEPENDENCIES for CVEs · pin versions · Dependabot ➕
IMAGE         tag :${{ github.sha }} · Trivy scan BEFORE push · severity HIGH,CRITICAL · exit-code 1
PUSH          docker/login-action@v3 with secrets · docker push · needs: [test, sast, sca] · if main
FIX CLASS     actions/checkout@v2 → v4 · setup-python@v2 → v5 · /greet vs /api/greet
DEBUG         pytest -v / -vv / -x / -s · Actions "Enable debug logging" / ACTIONS_STEP_DEBUG
```

---

## 14. Likely Exam / Interview Questions

1. **Walk through deploying an app manually, then with Docker, then with Kubernetes. What does each step add?** Manual is quick but not repeatable. Docker gives the same environment everywhere. Kubernetes adds replicas, self-healing, Services and rolling updates. CI/CD automates all of it.
2. **Why use a virtual environment for a Python project?** It isolates each project's packages and versions from the system and from other projects.
3. **A Flask app works locally but not in Docker with `-p 5001:5001`. Most likely cause?** It's bound to `127.0.0.1`. Use `host="0.0.0.0"`.
4. **Why copy `requirements.txt` before the rest of the code in the Dockerfile?** Layer caching: dependencies are only reinstalled when `requirements.txt` changes.
5. **A locally built image shows `ErrImagePull` on minikube. Why, and how do you fix it?** minikube has its own runtime and can't see the image, and the `:latest` tag forces a pull. Use `minikube image load` with `imagePullPolicy: IfNotPresent`, or push to a registry.
6. **Explain the port chain for a NodePort Service in front of the Flask app.** nodePort (30001) → Service port (80) → targetPort (5001) = containerPort, where Flask listens.
7. **What is DevSecOps, and what does "shift left" mean?** Building security checks into the pipeline. "Shift left" means finding issues earlier, when they're cheaper to fix.
8. **SAST vs. SCA vs. container scanning vs. DAST?** Your source code / your dependencies / the built image / the running app.
9. **What does CodeQL do, and where do its results appear?** SAST: it queries your code for vulnerability patterns. Results appear under the repo's Security → Code scanning alerts.
10. **What does pip-audit check?** Installed or required Python package versions against known-vulnerability databases.
11. **Why run Trivy before pushing the image?** So images with HIGH or CRITICAL CVEs fail the pipeline and are never published.
12. **Why tag images with the commit SHA?** It's unique and traceable to the exact code, it makes deploys reproducible, and rollback is easy.
13. **What does `--cov-fail-under=100` do? Does 100% coverage mean bug-free code?** It fails the pipeline if coverage drops below 100%. No: it means every line ran, not that every behaviour was checked.
14. **How do the test, SAST, SCA and Docker jobs relate in the workflow?** The first three run in parallel. The Docker job has `needs: [test, sast, sca]`, so it only runs if all of them pass, and only on `main`.
15. **How do you get more detail when a test or workflow fails?** `pytest -vv` (plus `-x` and `-s`), and GitHub Actions debug logging (re-run with debug, or `ACTIONS_STEP_DEBUG=true`).
