# CI/CD & DevSecOps: Advanced MCQs

> **Practice for [Lecture 15 – CI/CD with GitHub Actions](15-CICD-with-GitHub-Actions.md) and [Lecture 16 – End-to-End Deployment Project](16-End-to-End-Deployment-Project.md).** Each answer is shown right below its question. Cover it if you want to test yourself first.

| Part | Lecture | Questions |
|------|---------|-----------|
| [A](#part-a-lecture-15--cicd-with-github-actions) | 15 – CI/CD with GitHub Actions | Q1–Q8 |
| [B](#part-b-lecture-16--end-to-end-devsecops-project) | 16 – End-to-End DevSecOps Project | Q9–Q16 |

---

## Part A: Lecture 15 – CI/CD with GitHub Actions

### Q1. Files missing in the next job
Job `build` runs `./build.sh`, which creates `build/app.zip`. Job `deploy` has `needs: build` and runs `ls build/`, which fails with `No such file or directory`. Why?
- A) `needs:` runs both jobs at the same time
- B) Each job runs on a fresh runner, so `build/` only existed on the first machine. Upload it as an artifact in `build` and download it in `deploy`.
- C) Files are only kept between jobs on self-hosted runners with Windows
- D) `ls` isn't available on `ubuntu-latest`

> ✅ **Answer: B.** Jobs don't share a filesystem. `needs:` only controls **order**. To pass files, use `actions/upload-artifact@v4` → `actions/download-artifact@v4`. · 📖 [L15 §8](15-CICD-with-GitHub-Actions.md#8-artifacts)

### Q2. Two jobs, no `needs`
A workflow has jobs `test` and `deploy`, and **neither** has `needs:`. A commit breaks a test. What happens?
- A) `deploy` waits for `test` and is skipped
- B) Both start in parallel, so `deploy` can finish even though `test` fails
- C) `deploy` runs first because it's listed second
- D) GitHub refuses to run a workflow with two jobs and no dependency

> ✅ **Answer: B.** Jobs run **in parallel** by default. To deploy only after tests pass, add `needs: test` to `deploy`. · 📖 [L15 §4](15-CICD-with-GitHub-Actions.md#4-anatomy-of-a-workflow-file)

### Q3. An old artifact action
An old workflow uses `actions/upload-artifact@v3`. The run fails before doing any work. Most likely reason?
- A) The runner is out of disk space
- B) GitHub has shut down v1–v3 of the artifact actions, so the job fails automatically. Upgrade to `@v4`.
- C) Artifacts can only be uploaded from the `main` branch
- D) `upload-artifact` needs a secret called `ARTIFACT_TOKEN`

> ✅ **Answer: B.** Workflows that use the retired artifact action versions are failed automatically. The class YAML used `@v2`, so it has the same problem. · 📖 [L15 §7](15-CICD-with-GitHub-Actions.md#7-the-ci-workflow-corrected-class-example)

### Q4. A script that hides failures
`build.sh` has **no** `set -e`. It runs `pytest` (2 tests fail) and then `cp -r app build/` (succeeds). What does the "Run build" step show?
- A) ❌ Failed, because pytest failed
- B) ✅ Passed, because the step only sees the exit code of the **last** command
- C) ⚠️ A warning, but the job continues
- D) It hangs until it times out

> ✅ **Answer: B.** A shell script's exit code is that of its last command unless `set -e` stops it at the first failure. Without `set -e`, broken code passes CI. · 📖 [L15 §6](15-CICD-with-GitHub-Actions.md#6-hands-on-the-python-calculator-project)

### Q5. A manual approval before production
Every push to `main` is built, tested and deployed to staging automatically. Deploying to **production** waits for a team lead to click "Approve" (a GitHub Environment with required reviewers). Which practice is this?
- A) Continuous Integration only
- B) Continuous Delivery
- C) Continuous Deployment
- D) Blue-green deployment

> ✅ **Answer: B.** Delivery means every build is **ready** to release, with a **manual** step to production. Continuous **Deployment** would release to production automatically with no human step. · 📖 [L15 §2](15-CICD-with-GitHub-Actions.md#2-what-is-cicd)

### Q6. A secret in the logs
A step runs `echo "Token is ${{ secrets.API_TOKEN }}"`. What appears in the job log?
- A) The real token, in plain text
- B) `Token is ***`
- C) An error: secrets can't be used in `run:`
- D) `Token is ${{ secrets.API_TOKEN }}`

> ✅ **Answer: B.** GitHub masks known secret values in logs. Still, never echo secrets on purpose: masking can miss changed forms of the value (e.g. base64-encoded). · 📖 [L15 §9](15-CICD-with-GitHub-Actions.md#9-secrets)

### Q7. A pull request from a fork
The Docker login step works on your own pushes but fails on a **pull request from a forked repo**, because `secrets.DOCKERHUB_TOKEN` is empty. Why?
- A) The secret name is case-sensitive
- B) Secrets aren't passed to workflows triggered by PRs from forks, so strangers can't steal them
- C) Docker Hub blocks GitHub-hosted runners
- D) Fork PRs run on Windows runners

> ✅ **Answer: B.** This is deliberate protection: a fork author could otherwise change the workflow to print or send your secrets. Run publishing jobs only on `push` to `main`. · 📖 [L15 §9](15-CICD-with-GitHub-Actions.md#9-secrets)

### Q8. A "Run workflow" button
Which trigger adds a **"Run workflow" button** to the Actions tab so you can start the pipeline by hand?
- A) `on: push`
- B) `on: schedule`
- C) `on: workflow_dispatch`
- D) `on: pull_request`

> ✅ **Answer: C.** `workflow_dispatch` = manual trigger (with optional inputs). `schedule` uses cron for timed runs. · 📖 [L15 §5](15-CICD-with-GitHub-Actions.md#5-triggers-on)

---

## Part B: Lecture 16 – End-to-End DevSecOps Project

### Q9. Flask in Docker shows nothing
`app.py` ends with `app.run(port=5001)`. You run `docker run -p 5001:5001 python-web`, and `curl localhost:5001` gets `Connection reset` / empty reply, although the container is running. Why?
- A) Port 5001 can't be published on macOS
- B) Flask is listening on `127.0.0.1` **inside** the container, which only accepts connections from inside it. Use `host="0.0.0.0"`.
- C) `EXPOSE 5001` is missing, and that's what publishes the port
- D) The image needs `gunicorn` to accept any connection

> ✅ **Answer: B.** Inside a container, `127.0.0.1` means "this container only". `-p` forwards traffic to the container's network interface, so the app must listen on `0.0.0.0`. (`EXPOSE` is only documentation; `-p` does the publishing.) · 📖 [L16 §2](16-End-to-End-Deployment-Project.md#2-the-application), [L5 §9](05-Docker.md#9-port-mapping--docker-networking)

### Q10. minikube still pulls
You ran `minikube image load python-web:latest`. The Deployment uses `image: python-web:latest` with **no** `imagePullPolicy`. Pods show `ErrImagePull`. Why?
- A) `minikube image load` only works for images from Docker Hub
- B) For the `:latest` tag the default `imagePullPolicy` is `Always`, so Kubernetes tries to download it from Docker Hub. Set `imagePullPolicy: IfNotPresent`.
- C) minikube can't run Python images
- D) The Service is missing

> ✅ **Answer: B.** Images tagged `:latest` (or untagged) default to `Always`, and other tags default to `IfNotPresent`. The loaded image is ignored and the pull fails. · 📖 [L16 §5](16-End-to-End-Deployment-Project.md#5-stage-3-run-it-on-kubernetes-minikube)

### Q11. SAST or SCA?
The pipeline reports two problems: (1) `requirements.txt` pins `flask==2.2.0`, which has a known CVE, and (2) `app.py` passes user input into `os.system()`. Which tool finds which?
- A) CodeQL finds both
- B) (1) pip-audit (SCA), (2) CodeQL (SAST)
- C) (1) CodeQL (SAST), (2) pip-audit (SCA)
- D) Trivy finds both, and the others are redundant

> ✅ **Answer: B.** **SCA** checks the versions of **third-party dependencies** against vulnerability databases. **SAST** analyses **your own source code** for dangerous patterns like command injection. · 📖 [L16 §6](16-End-to-End-Deployment-Project.md#6-stage-4-the-devsecops-cicd-pipeline)

### Q12. Why scan before pushing?
Why does the Docker job run Trivy **before** `docker push`, not after?
- A) Trivy can only scan images that aren't in a registry
- B) So an image with HIGH/CRITICAL CVEs fails the job and is **never published** for anyone to deploy
- C) Pushing first would make the scan slower
- D) Docker Hub rejects images that haven't been scanned

> ✅ **Answer: B.** A gate is only useful if it comes before the release. With `exit-code: 1`, findings stop the job before the push step runs. · 📖 [L16 §10](16-End-to-End-Deployment-Project.md#10-pipeline-stage-build--trivy-scan--push)

### Q13. The Docker job on a pull request
The `docker` job has `needs: [test, sast, sca]` and `if: github.ref == 'refs/heads/main'`. You open a **pull request into `main`**, and all checks pass. Does the image get pushed?
- A) Yes, because the PR targets `main`
- B) No. On a pull request, `github.ref` is `refs/pull/<n>/merge`, so the `if:` is false and the job is skipped.
- C) Yes, but only the Trivy step runs
- D) No, because `needs:` jobs never run on PRs

> ✅ **Answer: B.** On PRs, `github.ref` isn't `refs/heads/main`. That's intended: PRs are **checked** (test, SAST, SCA), and only real pushes or merges to `main` **publish**. · 📖 [L16 §11](16-End-to-End-Deployment-Project.md#11-the-complete-workflow-file)

### Q14. A new route with no test
You add a new `/api/version` route but no test for it. The test job runs `pytest --cov=app --cov-fail-under=100`. What happens in the pipeline?
- A) Tests pass, coverage shows a warning, and the image is pushed
- B) The test job fails (coverage below 100%), and the Docker job doesn't run because it `needs` test
- C) Only the SCA job fails
- D) CodeQL adds the missing test automatically

> ✅ **Answer: B.** `--cov-fail-under` turns low coverage into a failing exit code. `--cov-report=term-missing` shows exactly which lines need a test. · 📖 [L16 §7](16-End-to-End-Deployment-Project.md#7-pipeline-stage-unit-tests--100-coverage)

### Q15. CodeQL permission error
The CodeQL `analyze` step fails with `Resource not accessible by integration`. What's missing?
- A) A `GITHUB_TOKEN` secret you must create by hand
- B) `permissions: security-events: write` on the job
- C) `python-version: 3.12` in `init`
- D) A Dockerfile

> ✅ **Answer: B.** CodeQL uploads its results to the repo's Code scanning alerts, which needs the `security-events: write` permission. `GITHUB_TOKEN` itself is created automatically. · 📖 [L16 §8](16-End-to-End-Deployment-Project.md#8-pipeline-stage-sast-with-codeql)

### Q16. pytest can't find the app
The repo has `app.py` in the root and tests in `tests/test_app.py` (`from app import app`). In CI, `pytest tests/` fails with `ModuleNotFoundError: No module named 'app'`. What's the simplest fix?
- A) Move `app.py` into `tests/`
- B) Run `python -m pytest tests/` from the repo root
- C) Rename `app.py` to `main.py`
- D) Install Flask globally with `sudo`

> ✅ **Answer: B.** `python -m pytest` adds the current folder (the repo root) to Python's import path, so `import app` works. (➕ An empty `conftest.py` in the root has the same effect.) · 📖 [L16 §12](16-End-to-End-Deployment-Project.md#12-troubleshooting--debugging)
