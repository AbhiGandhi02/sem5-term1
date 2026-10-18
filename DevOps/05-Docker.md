# Docker: Containers & Images

> **DevOps – Lecture 5 notes.** Anything marked **➕ Extra** was not covered in class. It's there because it fills a gap or comes up often in labs and interviews.
>
> **Builds on:** [Lecture 1 – Linux](01-Linux-Permissions-and-Commands.md) (processes, files), [Lecture 3 – Networking](03-Networking.md) (ports, `localhost` vs. `0.0.0.0`) and [Lecture 4 – Git](04-Git-Version-Control.md) (push your Dockerfiles to GitHub).
>
> **Next:** [Lecture 6 – Docker Part 2](06-Docker-Lifecycle-and-Multi-Stage.md) (full lifecycle, disk cleanup, multi-stage builds, security).
>
> **Practice:** [Docker MCQs](Docker-MCQs.md).

---

## Table of Contents
1. [The Big Picture: Why Docker?](#1-the-big-picture-why-docker)
2. [Containers vs. Virtual Machines](#2-containers-vs-virtual-machines)
3. [Docker Components](#3-docker-components)
4. [Images vs. Containers](#4-images-vs-containers)
5. [The Dockerfile](#5-the-dockerfile)
6. [Layers & Caching](#6-layers--caching)
7. [The Container Lifecycle](#7-the-container-lifecycle)
8. [Core Commands](#8-core-commands)
9. [Port Mapping & Docker Networking](#9-port-mapping--docker-networking)
10. [Registries: Docker Hub, ECR, ACR](#10-registries-docker-hub-ecr-acr)
11. [Docker Compose: Multi-Container Apps](#11-docker-compose-multi-container-apps)
12. [Hands-on: Dockerize a Node.js App (Homework)](#12-hands-on-dockerize-a-nodejs-app-homework)
13. [Quick Revision Cheat Sheet](#13-quick-revision-cheat-sheet)
14. [Likely Exam / Interview Questions](#14-likely-exam--interview-questions)

---

## 1. The Big Picture: Why Docker?

**The classic problem:** "It works on my machine!" The app runs on your laptop, but fails on the server because of a different OS, Node/Python version or missing library.

**Docker's answer:** pack the **app + all its dependencies** (runtime, libraries, config) into one box called a **container**. The same container runs the same way on **any** machine that has Docker.

> 💡 **Simple words mein:** Docker = shipping container. Andar kya hai (Node app, Python app) farak nahi padta, ship (server) pe har container same tarike se load hota hai.

**Why DevOps cares:** CI/CD pipelines build Docker images, cloud platforms (ECS, Kubernetes) run them, and every environment (dev → test → prod) uses the **same image**.

---

## 2. Containers vs. Virtual Machines

```
        VIRTUAL MACHINES                          CONTAINERS
 ┌────────┐ ┌────────┐ ┌────────┐       ┌────────┐ ┌────────┐ ┌────────┐
 │ App A  │ │ App B  │ │ App C  │       │ App A  │ │ App B  │ │ App C  │
 │ Libs   │ │ Libs   │ │ Libs   │       │ Libs   │ │ Libs   │ │ Libs   │
 │Guest OS│ │Guest OS│ │Guest OS│       └────────┘ └────────┘ └────────┘
 └────────┘ └────────┘ └────────┘       ┌──────────────────────────────┐
 ┌──────────────────────────────┐       │        Docker Engine         │
 │  Hypervisor (VMware, KVM)    │       ├──────────────────────────────┤
 ├──────────────────────────────┤       │   Host OS (shared KERNEL)    │
 │           Host OS            │       ├──────────────────────────────┤
 ├──────────────────────────────┤       │          Hardware            │
 │          Hardware            │       └──────────────────────────────┘
 └──────────────────────────────┘
```

| | Container | Virtual Machine |
|-|-----------|-----------------|
| OS | **Shares the host's kernel** | Full guest OS each |
| Size | MBs | GBs |
| Start time | Seconds (or less) | Minutes |
| Isolation | Process-level (lighter) | Hardware-level (stronger) |
| Best for | Many copies of an app, microservices, CI/CD | Running a different OS, strong isolation, legacy apps |

> 💡 **Simple words mein:** VM = alag-alag ghar, har ghar ki apni neev (OS). Container = ek building ke flats, neev (kernel) sabki same.
>
> **➕ Note:** since containers share the **Linux** kernel, Docker Desktop on Mac/Windows quietly runs a small Linux VM underneath.

---

## 3. Docker Components

```
  You ──▶ Docker CLI ──(REST API)──▶ Docker Daemon (dockerd) ──▶ images, containers, networks, volumes
          "docker run"                     │
                                           └── pull / push ──▶ Registry (Docker Hub, ECR, ACR)
```

| Component | What it is |
|-----------|------------|
| **Docker CLI** | The `docker` command you type. It just sends requests to the daemon. |
| **Docker Daemon** (`dockerd`) | Background service on the host that actually **builds, runs and manages** containers. |
| **Image** | Read-only template: code + runtime + libraries + env vars + config. |
| **Container** | A **running instance** of an image, isolated from other containers. |
| **Dockerfile** | Text file with the steps to **build** an image. |
| **Registry** | Online store for images (Docker Hub by default). |

> ➕ `Cannot connect to the Docker daemon` = the daemon isn't running. Start Docker Desktop, or `sudo systemctl start docker` on Linux.
> ➕ `permission denied ... docker.sock` on Linux → `sudo usermod -aG docker $USER`, then log out and back in.

---

## 4. Images vs. Containers

| Image | Container |
|-------|-----------|
| **Blueprint / template** | **Running instance** of it |
| Read-only | Adds a thin **writable layer** on top |
| Built once with `docker build` | Many can be started from one image with `docker run` |
| Stored on disk / in a registry | Has a state: running, stopped, … |

> 💡 **Simple words mein:** image = class, container = object. Ek class se kitne bhi objects bana lo.
>
> ⚠️ Data written inside a container is **lost when the container is removed**. ➕ Keep important data in a **volume**: `docker run -v mydata:/var/lib/mysql ...`

**Image names:** `name:tag`, e.g. `node:20-alpine`, `nginx:latest`, `myuser/myapp:1.0`. No tag means `:latest`.

---

## 5. The Dockerfile

A **Dockerfile** is a recipe Docker follows, top to bottom, to build an image.

**Example: a Node.js app**
```dockerfile
FROM node:20-alpine            # base image to start from
WORKDIR /app                   # cd into /app (creates it if missing)
COPY package*.json ./          # copy dependency list first (for caching, see §6)
RUN npm install                # runs at BUILD time, result saved in the image
COPY . .                       # copy the rest of the source code
EXPOSE 3000                    # documents the port the app listens on
CMD ["node", "server.js"]      # default command when a CONTAINER starts
```

| Instruction | Purpose |
|-------------|---------|
| `FROM` | Base image. Always the first instruction. |
| `WORKDIR` | Set the working directory for the following steps |
| `COPY` | Copy files from your machine (build context) into the image |
| `RUN` | Run a command **while building** (install packages) |
| `ENV` ➕ | Set an environment variable |
| `EXPOSE` | Document which port the app uses (doesn't publish it, `-p` does) |
| `CMD` | Default command at **container start**. Easy to override: `docker run img <other cmd>` |
| `ENTRYPOINT` | Fixed command at container start. `CMD` then supplies its default arguments. |

**`RUN` vs. `CMD`:** `RUN` happens at **build time** (once). `CMD` happens at **run time** (every container start).

**➕ `.dockerignore`:** like `.gitignore`, keeps files out of the build:
```
node_modules
.git
.env
*.log
```

---

## 6. Layers & Caching

Each instruction in a Dockerfile (`FROM`, `RUN`, `COPY`, …) creates a **layer**. An image is a **stack of read-only layers**.

```
 ┌───────────────────────────┐  ← container's writable layer (docker run)
 ├───────────────────────────┤
 │ COPY . .                  │  ← changes on every code edit
 │ RUN npm install           │  ← reused from cache if package.json unchanged ✅
 │ COPY package*.json ./     │
 │ WORKDIR /app              │
 │ FROM node:20-alpine       │  ← shared by every image built on node:20-alpine
 └───────────────────────────┘
```

**Caching rule:** on a rebuild, Docker **reuses** each layer whose instruction and files haven't changed. **Once one layer changes, every layer after it is rebuilt.**

> 💡 **Simple words mein:** jo cheez kam badalti hai (dependencies) usse upar rakho, jo roz badalti hai (code) usse neeche. Tab `npm install` har baar dobara nahi chalega.

**That's why** the Dockerfile copies `package.json` and runs `npm install` **before** `COPY . .`. Editing code then only rebuilds the last layer.

```bash
docker history myapp     # ➕ see each layer and its size
docker build --no-cache -t myapp .   # ➕ force a full rebuild
```

---

## 7. The Container Lifecycle

```
 Dockerfile ──build──▶ Image ──run──▶ Running ──stop──▶ Stopped ──rm──▶ (gone)
                         ▲               ▲                  │
                       pull              └──── start ───────┘
```

| Step | Command | What happens |
|------|---------|--------------|
| **Build** | `docker build -t myapp .` | Dockerfile → image |
| **Run** | `docker run myapp` | Creates **and** starts a container |
| **Stop** | `docker stop <id/name>` | Graceful stop (SIGTERM, then SIGKILL after 10 s) |
| **Start** ➕ | `docker start <id/name>` | Restart a stopped container |
| **Remove** | `docker rm <id/name>` | Delete a **stopped** container (`-f` to force a running one) |
| **Remove image** ➕ | `docker rmi myapp` | Delete the image |

---

## 8. Core Commands

### Build
```bash
docker build -t myapp:1.0 .      # -t = name:tag   . = build context (folder with the Dockerfile)
docker images                    # list local images
```

### Run
```bash
docker run -d -p 8080:3000 --name web myapp:1.0
```
| Flag | Meaning |
|------|---------|
| `-d` | **Detached**: run in the background, give the terminal back |
| `-p 8080:3000` | Map **host port 8080 → container port 3000** |
| `--name web` | Friendly name instead of a random one |
| `-e KEY=val` ➕ | Set an environment variable |
| `-it` ➕ | Interactive terminal, e.g. `docker run -it ubuntu bash` |
| `--rm` ➕ | Auto-delete the container when it stops |

### Inspect & manage
```bash
docker ps                  # RUNNING containers
docker ps -a               # ALL containers, including stopped
docker stop web            # stop
docker rm web              # remove
docker logs -f web         # ➕ follow the app's output (first thing to check when it crashes)
docker exec -it web sh     # ➕ open a shell INSIDE a running container
docker pull nginx          # ➕ download an image from the registry
docker system prune        # ➕ clean up stopped containers, unused networks, dangling images
```

> **Debug order when a container won't work:** `docker ps -a` (did it exit?) → `docker logs <name>` (why?) → `docker exec -it <name> sh` (look inside).

---

## 9. Port Mapping & Docker Networking

### Port mapping
A container has its **own network**. Its ports are **not reachable from outside** until you publish them with `-p HOST:CONTAINER`.

```
 Browser ──▶ localhost:8080 (host) ──▶ -p 8080:3000 ──▶ container :3000 (Node app)
```
```bash
docker run -d -p 8080:80 nginx    # open http://localhost:8080 → nginx's welcome page
```
> ⚠️ Inside the container, the app must listen on **`0.0.0.0`**, not `127.0.0.1`. Otherwise it only accepts connections from inside the container itself (see [Networking §2](03-Networking.md#2-ip-addresses)).
>
> On a cloud server, also **open the host port** in the security group / firewall.

### Container-to-container networking
Containers on the **same user-defined network** can reach each other **by container name** (Docker runs a small DNS for them).

```bash
docker network create mynet
docker run -d --name db  --network mynet -e POSTGRES_PASSWORD=secret postgres
docker run -d --name api --network mynet -p 3000:3000 myapi
# inside "api", the database host is simply:  db:5432
```

| ➕ Network driver | Use |
|------------------|-----|
| `bridge` (default) | Containers on one host |
| `host` | Container uses the host's network directly (no `-p` needed, Linux only) |
| `none` | No network |

---

## 10. Registries: Docker Hub, ECR, ACR

A **registry** stores and distributes images, like GitHub but for images.

| Registry | Provider |
|----------|----------|
| **Docker Hub** | Default public registry (`docker pull nginx` comes from here) |
| **Amazon ECR** | AWS private registry |
| **Azure ACR** | Azure private registry |
| ➕ GHCR | GitHub Container Registry |

**Push your image to Docker Hub:**
```bash
docker login
docker tag myapp:1.0 <dockerhub-user>/myapp:1.0    # name must start with your username
docker push <dockerhub-user>/myapp:1.0
# anyone, anywhere:  docker pull <dockerhub-user>/myapp:1.0
```

---

## 11. Docker Compose: Multi-Container Apps

Real apps need several containers (web + API + DB). Typing many `docker run` commands is painful. **Docker Compose** describes them all in **one YAML file** and starts them with **one command**.

**`docker-compose.yml`**
```yaml
services:
  web:
    build: .                  # build from the Dockerfile in this folder
    ports:
      - "8080:3000"
    environment:
      - DB_HOST=db            # service name works as a hostname
    depends_on:
      - db
  db:
    image: postgres:16
    environment:
      - POSTGRES_PASSWORD=secret
    volumes:
      - dbdata:/var/lib/postgresql/data   # data survives restarts

volumes:
  dbdata:
```

```bash
docker compose up -d       # build (if needed) + start everything in the background
docker compose ps          # status
docker compose logs -f     # logs of all services
docker compose down        # stop and remove containers + network (add -v to delete volumes)
```
> Compose automatically puts all services on **one shared network**, so `web` reaches the database at `db:5432`.

---

## 12. Hands-on: Dockerize a Node.js App (Homework)

**1. `server.js`**
```js
const http = require("http");
http.createServer((req, res) => {
  res.writeHead(200, { "Content-Type": "text/html" });
  res.end("<h1>Hello from Docker 🐳</h1>");
}).listen(3000, "0.0.0.0", () => console.log("Listening on 3000"));
```

**2. `package.json`**
```json
{ "name": "docker-demo", "version": "1.0.0", "main": "server.js" }
```

**3. `Dockerfile`**: use the one from [§5](#5-the-dockerfile).

**4. Build, run, test**
```bash
docker build -t docker-demo .
docker run -d -p 8080:3000 --name demo docker-demo
curl localhost:8080              # or open http://localhost:8080
docker logs demo                 # "Listening on 3000"
```

**5. Push to GitHub** (see [Lecture 4](04-Git-Version-Control.md#13-team-workflow-feature-branches--pull-requests))
```bash
git init && git add . && git commit -m "Dockerize Node.js demo app"
git remote add origin git@github.com:<user>/docker-demo.git
git push -u origin main
```

**Try next:** dockerize a small Python Flask app the same way (`FROM python:3.12-slim`, `RUN pip install -r requirements.txt`, `CMD ["python", "app.py"]`), then run both apps together with Compose.

---

## 13. Quick Revision Cheat Sheet

```
WHY          app + dependencies in one box → runs the same everywhere
VS VM        container shares host KERNEL (MBs, seconds)   VM = full guest OS (GBs, minutes)
PARTS        CLI → daemon (dockerd) → images/containers    registry = Docker Hub / ECR / ACR
IMAGE        read-only template (class)     CONTAINER = running instance (object) + writable layer
DOCKERFILE   FROM  WORKDIR  COPY  RUN(build time)  ENV  EXPOSE  CMD / ENTRYPOINT(run time)
LAYERS       1 instruction = 1 layer; unchanged layers cached; a change rebuilds everything below
             → copy package.json + install deps BEFORE copying code
LIFECYCLE    build → run → stop → (start) → rm             rmi = delete image
BUILD        docker build -t name:tag .                    docker images
RUN          docker run -d -p HOST:CONTAINER --name n img   -e  -it  --rm  -v
LIST         docker ps (running)    docker ps -a (all)
DEBUG        docker logs -f n    docker exec -it n sh    app must listen on 0.0.0.0
NETWORK      docker network create net; same network → reach by container NAME
REGISTRY     docker login → docker tag img user/img:tag → docker push
COMPOSE      docker-compose.yml → docker compose up -d / ps / logs / down
```

---

## 14. Likely Exam / Interview Questions

1. **What is Docker and what problem does it solve?** It packages an app with its dependencies so it runs the same everywhere ("works on my machine").
2. **Container vs. VM?** Containers share the host kernel and are light and fast. VMs run a full guest OS on a hypervisor and are heavier, with stronger isolation.
3. **Image vs. container?** An image is a read-only template. A container is a running instance of it with a writable layer.
4. **What does the Docker daemon do?** It runs in the background and builds, runs and manages containers. The CLI just sends it commands.
5. **`RUN` vs. `CMD`?** `RUN` executes at build time. `CMD` is the default command at container start.
6. **`CMD` vs. `ENTRYPOINT`?** `CMD` is easily overridden by `docker run` arguments. `ENTRYPOINT` stays fixed, and `CMD` supplies its default arguments.
7. **How does layer caching work, and how do you order a Dockerfile for it?** Unchanged layers are reused, and a change rebuilds all layers after it. Put dependency installs before copying source code.
8. **What do `-d` and `-p 8080:3000` do?** `-d` runs in the background. `-p` maps host port 8080 to container port 3000.
9. **`docker ps` vs. `docker ps -a`?** Running containers only, vs. all containers.
10. **`docker stop` vs. `docker rm`?** `stop` halts the container (it can be started again). `rm` deletes a stopped container.
11. **App runs in the container but the browser can't reach it. Why?** No `-p` mapping, the app is bound to `127.0.0.1` instead of `0.0.0.0`, or the firewall/security group blocks the host port.
12. **How do two containers talk to each other?** Put them on the same user-defined network (or in one Compose file) and use the container/service name as the hostname.
13. **What is a registry? Name some.** A store for images: Docker Hub, Amazon ECR, Azure ACR, GHCR.
14. **What is Docker Compose?** A YAML file describing a multi-container app, started with `docker compose up`.
15. **What happens to data inside a container when it's removed? How do you keep it?** It's lost. Use volumes (`-v`).
