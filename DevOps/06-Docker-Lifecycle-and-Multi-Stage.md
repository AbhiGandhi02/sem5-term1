# Docker Part 2: Lifecycle, Storage & Multi-Stage Builds

> **DevOps – Lecture 6 notes.** Anything marked **➕ Extra** was not covered in class. It's there because it fills a gap or comes up often in labs and interviews.
>
> **Builds on:** [Lecture 5 – Docker](05-Docker.md). Basics like images vs. containers, Dockerfile instructions, layers, ports and Compose are **not repeated here**. This lecture goes deeper.
>
> **Next:** [Lecture 7 – Docker Networking & Volumes](07-Docker-Networking-and-Volumes.md).
>
> **Practice:** [Docker MCQs](Docker-MCQs.md).

---

## Table of Contents
1. [Quick Recap](#1-quick-recap)
2. [Why Docker? (the bigger reasons)](#2-why-docker-the-bigger-reasons)
3. [The Full Container Lifecycle](#3-the-full-container-lifecycle)
4. [Working with Images: `pull`, `images`, `rmi`](#4-working-with-images-pull-images-rmi)
5. [Looking Inside: `docker exec`](#5-looking-inside-docker-exec)
6. [Common Errors & Fixes](#6-common-errors--fixes)
7. [Managing Disk Space: `system df` & `prune`](#7-managing-disk-space-system-df--prune)
8. [Multi-Stage Builds](#8-multi-stage-builds)
9. [Docker & DevSecOps: Secure Images](#9-docker--devsecops-secure-images)
10. [Quick Revision Cheat Sheet](#10-quick-revision-cheat-sheet)
11. [Likely Exam / Interview Questions](#11-likely-exam--interview-questions)

---

## 1. Quick Recap

| Term | One line | More in |
|------|----------|---------|
| **Container** | Lightweight, standalone package: code + runtime + libraries + env vars | [L5 §4](05-Docker.md#4-images-vs-containers) |
| **Image** | Read-only **blueprint** used to create containers | [L5 §4](05-Docker.md#4-images-vs-containers) |
| **Dockerfile** | Instructions to build an image: `docker build -t myapp .` | [L5 §5](05-Docker.md#5-the-dockerfile) |

> 💡 **Simple words mein:** image = building ka naksha (blueprint), container = us naksha se bani asli building. Ek naksha, kitni bhi buildings.

---

## 2. Why Docker? (the bigger reasons)

| Reason | What it means |
|--------|---------------|
| **Same everywhere** | The image that passed tests in dev is the **exact** image that runs in prod |
| **Dependency management** | Each app carries its own versions, so Node 18 and Node 20 apps can run on one server |
| **Easy scaling** | Need more capacity? Start 10 more containers from the same image in seconds |
| **Microservices** | Each service (auth, payments, orders) lives in its own container and is deployed separately |
| **DevOps / CI-CD** | Pipelines build an image once, then test, scan and deploy that same image |

---

## 3. The Full Container Lifecycle

```
                docker create          docker start
   Image ───────────────────▶ CREATED ─────────────▶ RUNNING ◀────────────┐
     │                                        │  ▲        │                │
     └──────── docker run (create + start) ───┘  │        │ docker pause   │ docker unpause
                                                 │        ▼                │
                                    docker start │      PAUSED ────────────┘
                                                 │
                         docker stop / kill      │
              RUNNING ───────────────────▶ STOPPED (Exited)
                                                 │
                                       docker rm ▼
                                              REMOVED (gone, its data too)
```

| State | Meaning | Get there with |
|-------|---------|----------------|
| **Created** | Container exists but has **never started** | `docker create img` |
| **Running** | Main process is executing | `docker run img` / `docker start c` |
| **Paused** | Processes **frozen in memory** (not using CPU), resume instantly | `docker pause c` → `docker unpause c` |
| **Stopped** (Exited) | Main process ended. Container and its files **still exist** | `docker stop c` / app finished or crashed |
| **Removed** | Container **and its writable layer deleted** | `docker rm c` |

**`stop` vs. `kill` vs. `pause`:**

| Command | What it does |
|---------|--------------|
| `docker stop` | **Graceful**: sends SIGTERM, waits 10 s, then SIGKILL. The app can save work and close connections. |
| `docker kill` ➕ | **Immediate** SIGKILL. Use only when `stop` hangs. |
| `docker pause` | Freezes processes. Nothing exits, memory stays as it was. |
| `docker restart` ➕ | `stop` + `start` |

> 💡 **Simple words mein:** stop = app ko bolo "band ho jao" aur thoda time do. kill = seedha plug kheench do. pause = video pause, jahan tha wahin se aage chalega.

**See the state** in the `STATUS` column:
```bash
$ docker ps -a
CONTAINER ID  IMAGE  STATUS                     NAMES
3f2a1b...     nginx  Up 5 minutes               web
9c8d7e...     nginx  Up 2 minutes (Paused)      web2
1a2b3c...     myapp  Exited (0) 1 hour ago      job      ← 0 = finished normally
4d5e6f...     myapp  Exited (137) 3 min ago     api      ← 137 = killed (SIGKILL / out of memory)
7g8h9i...     redis  Created                    cache
```
> ➕ **Exit codes:** `0` = success, `1` = app error (check `docker logs`), `137` = killed (often **out of memory**), `143` = stopped by SIGTERM.

---

## 4. Working with Images: `pull`, `images`, `rmi`

```bash
docker pull nginx              # download nginx:latest from Docker Hub
docker pull node:20-alpine     # a specific tag
docker images                  # list local images (REPOSITORY, TAG, IMAGE ID, SIZE)
docker rmi nginx               # delete an image
docker inspect nginx           # ➕ full JSON details: layers, env, exposed ports
```

- `docker pull` fetches **the tag you ask for**. With no tag it pulls `:latest`.
- `docker run` **pulls automatically** if the image isn't on your machine yet.

> ⚠️ **`latest` is just a tag name**, not a promise of the newest version. In real deployments, **pin a version** (`node:20.11-alpine`) so a rebuild next month doesn't silently change your app.

---

## 5. Looking Inside: `docker exec`

Run a command **inside a running container**. Used for debugging, checking files and quick maintenance.

```bash
docker exec -it web sh            # open an interactive shell (use bash if the image has it)
docker exec web ls /usr/share/nginx/html     # run one command and exit
docker exec web cat /etc/os-release          # which OS is the base image?
docker exec -it db psql -U postgres          # open a DB client inside the DB container
```
| Flag | Meaning |
|------|---------|
| `-i` | Keep input open (so you can type) |
| `-t` | Give a terminal (prompt, colours) |

**`exec` vs. `run`:** `run` creates a **new** container. `exec` goes **into an existing, running** one.

> ⚠️ Changes made with `exec` (e.g. `apt install`) are **lost when the container is removed**. Permanent changes belong in the **Dockerfile**: rebuild the image.

---

## 6. Common Errors & Fixes

| Error message | Cause | Fix |
|---------------|-------|-----|
| `You cannot remove a running container ... Stop the container before attempting removal or force remove` | `docker rm` on a running container | `docker stop c && docker rm c`, or `docker rm -f c` |
| `conflict: unable to remove image ... image is being used by container` ➕ | A container (even stopped) still uses that image | Remove the container first, or `docker rmi -f` |
| `Bind for 0.0.0.0:8080 failed: port is already allocated` ➕ | Another container/app already uses host port 8080 | Use another host port (`-p 8081:3000`) or stop the other one |
| `Conflict. The container name "/web" is already in use` ➕ | A container with that name exists (maybe stopped) | `docker rm web`, or pick a new `--name` |
| `Cannot connect to the Docker daemon` ➕ | Daemon not running | Start Docker Desktop / `sudo systemctl start docker` |
| `no space left on device` ➕ | Old images, containers and build cache filled the disk | See [§7](#7-managing-disk-space-system-df--prune) |

**Why can't you `rm` a running container?** The **daemon** protects it: deleting it mid-run could corrupt data the app is writing. So Docker makes you stop it first (or say `-f` on purpose).

---

## 7. Managing Disk Space: `system df` & `prune`

Images, stopped containers, volumes and build cache **pile up** and can quietly eat tens of GBs.

### Check usage
```bash
$ docker system df
TYPE            TOTAL   ACTIVE   SIZE      RECLAIMABLE
Images          15      3        4.2GB     3.6GB (85%)
Containers      8       2        120MB     95MB (79%)
Local Volumes   4       1        1.1GB     800MB (72%)
Build Cache     42      0        2.3GB     2.3GB
```
**RECLAIMABLE** = space you can free because nothing is using it.

### Clean up
```bash
docker system prune            # stopped containers + unused networks + dangling images + build cache
docker system prune -a         # ALSO every image not used by a container ⚠️ (you'll re-download them)
docker system prune --volumes  # ALSO unused volumes ⚠️ (database data can be lost!)

# ➕ targeted versions
docker container prune         # stopped containers only
docker image prune             # dangling images only
docker builder prune           # build cache only
```

> 💡 **Dangling image** = an image with no name (`<none>:<none>`). It's left behind when you rebuild `myapp` and the tag moves to the new image.

---

## 8. Multi-Stage Builds

### The problem
To **build** an app you need heavy tools (compilers, `npm` dev dependencies, Maven, JDK). To **run** it you only need the output (`dist/`, a `.jar`, a binary). A normal Dockerfile ships **both**, so the image is huge and full of tools an attacker could use.

### The idea
Use **two (or more) `FROM`s in one Dockerfile**:
1. **Builder stage**: has all build tools, compiles the app, produces `dist/` or `target/`.
2. **Runtime stage**: starts from a small fresh image and **copies only the output** from the builder.

Everything in the builder stage is **thrown away**. Only the last stage becomes your image.

> 💡 **Simple words mein:** builder = kitchen (bartan, gas, sab saaman). Runtime = sirf plate mein khana leke customer ko do. Poori kitchen customer ko nahi bhejte.

### Example 1: a React / TypeScript frontend → Nginx
```dockerfile
# ---------- Stage 1: builder ----------
FROM node:20 AS builder            # "AS builder" names this stage
WORKDIR /app
COPY package*.json ./
RUN npm install                    # includes dev dependencies (TypeScript, Vite, ...)
COPY . .
RUN npm run build                  # produces /app/dist (static HTML/JS/CSS)

# ---------- Stage 2: runtime ----------
FROM nginx:alpine                  # tiny web server, no Node at all
COPY --from=builder /app/dist /usr/share/nginx/html   # take ONLY the build output
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### Example 2: a Java (Maven) app → `target/` folder
```dockerfile
FROM maven:3.9-eclipse-temurin-21 AS builder
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline      # cache dependencies as their own layer
COPY src ./src
RUN mvn package -DskipTests        # produces target/app.jar

FROM eclipse-temurin:21-jre-alpine # JRE only (runs Java), no JDK, no Maven
COPY --from=builder /app/target/*.jar /app/app.jar
CMD ["java", "-jar", "/app/app.jar"]
```

### The result

| | Single-stage | Multi-stage |
|-|--------------|-------------|
| React app | ~1.1 GB (`node:20` + node_modules) | **~45 MB** (`nginx:alpine` + `dist/`) |
| Java app | ~800 MB (Maven + JDK) | **~200 MB** (JRE + jar) |
| Build tools in prod image? | Yes ❌ | No ✅ |
| Pull / deploy speed | Slow | Fast |

*(Sizes are typical, yours will vary.)*

**Key syntax:**
- `FROM <image> AS <name>` → name a stage
- `COPY --from=<name> <src> <dest>` → copy files out of an earlier stage
- ➕ `docker build --target builder -t myapp:debug .` → build only up to a given stage

---

## 9. Docker & DevSecOps: Secure Images

**DevSecOps** = putting **security checks inside** the DevOps pipeline, not leaving them to the end. For Docker, that means **checking every image's layers for known vulnerabilities (CVEs)** before it is deployed.

### Scan images
```bash
docker scout cves myapp:1.0        # Docker's built-in scanner
trivy image myapp:1.0              # ➕ popular open-source scanner, common in CI pipelines
docker history myapp:1.0           # see each layer and the command that made it
```
In CI, the pipeline **fails the build** if a HIGH or CRITICAL vulnerability is found.

### Best practices
| Practice | Why |
|----------|-----|
| Use **small base images** (`alpine`, `-slim`, ➕ distroless) | Fewer packages = fewer vulnerabilities, smaller image |
| Use **multi-stage builds** | No compilers or build tools in production |
| **Pin versions** (`node:20.11-alpine`, not `latest`) | Repeatable builds, no surprise changes |
| ➕ Run as a **non-root user** (`USER node`) | If the app is hacked, the attacker isn't root |
| **Never put secrets in a Dockerfile** (`ENV API_KEY=...`, `COPY .env`) | Anyone with the image can read them with `docker history` / `inspect` |
| Use **`.dockerignore`** | Keeps `.git`, `.env` and `node_modules` out of the image |
| **One process per container** | Easier to scale, log and restart |
| **Rebuild regularly** | Picks up security patches in the base image |

---

## 10. Quick Revision Cheat Sheet

```
WHY           same image dev → prod · own dependencies · scale fast · microservices · CI/CD
IMAGE         read-only blueprint          CONTAINER = running instance of it
LIFECYCLE     Created → Running ⇄ Paused → Stopped (Exited) → Removed
              create · start · run(=create+start) · pause/unpause · stop(graceful) · kill · restart · rm
EXIT CODES    0 ok · 1 app error · 137 killed / OOM · 143 SIGTERM
IMAGES        docker pull img:tag (default :latest)  docker images  docker rmi  docker inspect
EXEC          docker exec -it c sh      run = NEW container, exec = INTO a running one
ERRORS        rm running → stop first or rm -f      rmi in use → rm container first
              port already allocated → other host port      name in use → rm old / new name
DISK          docker system df (RECLAIMABLE)
              docker system prune  [-a all unused images]  [--volumes ⚠️ data]
              container / image / builder prune      dangling = <none>:<none>
MULTI-STAGE   FROM big AS builder → build → FROM small → COPY --from=builder /app/dist .
              only the LAST stage ships · smaller · faster · no build tools in prod
DEVSECOPS     scan layers: docker scout cves / trivy image · small base · pin tags
              non-root USER · no secrets in image · .dockerignore · rebuild for patches
```

---

## 11. Likely Exam / Interview Questions

1. **Name the stages of a container's lifecycle.** Created → Running → Paused → Stopped (Exited) → Removed.
2. **`docker stop` vs. `docker kill` vs. `docker pause`?** Graceful SIGTERM vs. immediate SIGKILL vs. freezing processes in memory.
3. **`docker run` vs. `docker create` + `docker start`?** `run` does both steps in one command.
4. **What happens if you `docker rm` a running container?** Error. Stop it first, or use `docker rm -f`.
5. **What does `docker pull` do, and what tag does it use by default?** It downloads an image from a registry. The default tag is `:latest`.
6. **Why shouldn't you rely on `latest` in production?** It's just a tag that can point to a different image later, so builds aren't repeatable.
7. **What is `docker exec` used for? How is it different from `docker run`?** It runs a command inside an already running container. `run` starts a new one.
8. **Your server says "no space left on device". What do you do?** `docker system df` to see what's using space, then `docker system prune` (`-a` if needed, and be careful with `--volumes`).
9. **What is a dangling image?** An untagged `<none>:<none>` image left behind by a rebuild.
10. **What is a multi-stage build and why use it?** Several `FROM`s in one Dockerfile. Build in a heavy stage, copy only the output into a small runtime stage. You get smaller, faster and more secure images.
11. **What does `COPY --from=builder` do?** It copies files from an earlier named stage into the current one.
12. **What goes in the builder stage vs. the runtime stage?** Builder: compilers, dev dependencies, source code, the build command. Runtime: just the runtime plus `dist/`, `target/*.jar` or the binary.
13. **How does Docker fit into DevSecOps?** Images are scanned layer by layer for CVEs in the CI pipeline, and the build fails on serious issues.
14. **List three Dockerfile security best practices.** Small or pinned base images, a non-root user, no secrets in the image (plus multi-stage builds and `.dockerignore`).
15. **A container shows `Exited (137)`. What probably happened?** It was killed by SIGKILL, most often because it ran out of memory.
