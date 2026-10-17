# Docker Part 3: Networking & Volumes

> **DevOps – Lecture 7 notes.** Anything marked **➕ Extra** was not covered in class. It's there because it fills a gap or comes up often in labs and interviews.
>
> **Builds on:** [Lecture 5 – Docker](05-Docker.md) (port mapping, a first look at networks) and [Lecture 6 – Docker Part 2](06-Docker-Lifecycle-and-Multi-Stage.md) (lifecycle, `exec`, `prune`). Networking ideas (IP, DNS, ports) come from [Lecture 3](03-Networking.md).
>
> **Next:** [Lecture 8 – Kubernetes Basics](08-Kubernetes-Basics.md).

---

## Table of Contents
1. [Recap: The Docker Daemon](#1-recap-the-docker-daemon)
2. [Docker Networking: The Big Picture](#2-docker-networking-the-big-picture)
3. [The Four Network Types](#3-the-four-network-types)
4. [Network Commands](#4-network-commands)
5. [Lab: Two Containers Talking on a Bridge](#5-lab-two-containers-talking-on-a-bridge)
6. [Lab: Isolation with the `none` Network](#6-lab-isolation-with-the-none-network)
7. [Docker Storage: Why Volumes?](#7-docker-storage-why-volumes)
8. [Named Volumes vs. Bind Mounts](#8-named-volumes-vs-bind-mounts)
9. [Volume Commands](#9-volume-commands)
10. [Lab: Data That Survives a Container](#10-lab-data-that-survives-a-container)
11. [Lab: Live-Editing a Website with a Bind Mount](#11-lab-live-editing-a-website-with-a-bind-mount)
12. [Quick Revision Cheat Sheet](#12-quick-revision-cheat-sheet)
13. [Likely Exam / Interview Questions](#13-likely-exam--interview-questions)

---

## 1. Recap: The Docker Daemon

```
  docker CLI ──(REST API)──▶  Docker Daemon (dockerd)
                                 ├── images
                                 ├── containers
                                 ├── NETWORKS   ← this lecture
                                 └── VOLUMES    ← this lecture
```

- The **daemon** (`dockerd`) is the background service that **creates and manages** containers, images, networks and volumes.
- The CLI only sends it requests. **No daemon = no Docker:** every command fails with `Cannot connect to the Docker daemon`.

---

## 2. Docker Networking: The Big Picture

Every container gets its **own network stack**: its own IP, interfaces and ports. The network it joins decides **who it can talk to**.

```
                  HOST (your laptop / EC2)
   ┌──────────────────────────────────────────────────────┐
   │   eth0 (host IP, e.g. 172.31.5.10) ◀── internet       │
   │      │                                                │
   │   docker0 / my_bridge  (virtual switch, 172.18.0.1)   │
   │      ├── container1  172.18.0.2                       │
   │      └── container2  172.18.0.3                       │
   └──────────────────────────────────────────────────────┘
```

> 💡 **Simple words mein:** bridge network = ghar ke andar ka Wi-Fi router. Containers us router se jude devices hain. Aapas mein baat kar sakte hain, aur bahar (internet) bhi router ke through ja sakte hain (NAT).

---

## 3. The Four Network Types

| Driver | Scope | Can containers talk to each other? | Internet access? | Typical use |
|--------|-------|-----------------------------------|------------------|-------------|
| **bridge** (default) | One host | ✅ (same bridge) | ✅ via NAT | Most single-server apps |
| **host** | One host | Uses host's network directly | ✅ | Max network speed, no port mapping |
| **none** | One container | ❌ | ❌ | Fully isolated jobs |
| **overlay** | **Many hosts** | ✅ across machines | ✅ | Docker Swarm, distributed microservices |

### 3.1 Bridge (default)
- A **virtual switch** on the host. A container with no `--network` flag joins the default bridge, called **`bridge`** (`docker0`).
- Containers on the **same** bridge can communicate. Containers on **different** bridges can't.
- To reach a container from **outside**, publish a port with `-p 8080:80`.

> ⚠️ **Default bridge vs. your own bridge** ➕
> | | Default `bridge` | User-defined (`docker network create`) |
> |-|------------------|------------------------------------------|
> | Reach others **by name** (`ping container2`) | ❌ IP only | ✅ Built-in DNS |
> | Isolation from unrelated containers | ❌ Everyone shares it | ✅ Only containers you add |
>
> **Always create your own network** for apps that need to talk to each other. Compose does this for you automatically.

### 3.2 Host
```bash
docker run -d --network host nginx      # nginx is now on the HOST's port 80 directly
```
- **No separate network stack.** The container uses the host's IP and ports.
- `-p` is **ignored** (there's nothing to map). If port 80 is already taken on the host, the app fails.
- Faster (no NAT), but **less isolated**.
- ➕ Works fully on **Linux**. On Docker Desktop (Mac/Windows) it attaches to the hidden Linux VM, not your laptop.

> 💡 **Simple words mein:** host network = container ne apna alag phone number nahi liya, host ka hi number use kar raha hai.

### 3.3 None
```bash
docker run -d --network none alpine sleep 1000
```
- Only a **loopback** interface (`lo`, `127.0.0.1`). No IP, no internet, no other containers.
- **Use for:** batch jobs on sensitive data, untrusted code, or anything that must never "phone home".

### 3.4 Overlay
- **Spans multiple Docker hosts.** Containers on server A and server B join one virtual network and talk as if they were side by side.
- Needs **Docker Swarm** (or another orchestrator) to coordinate the hosts:
```bash
docker swarm init                                   # ➕ on the manager node
docker network create -d overlay --attachable my_overlay
```
- ➕ Kubernetes solves the same problem with its own networking plugins (CNI), so you'll meet the idea again there.

---

## 4. Network Commands

```bash
docker network create my_net            # new bridge network (bridge is the default driver)
docker network create -d overlay my_ov  # ➕ choose a driver with -d
docker network ls                       # list networks
docker network inspect my_net           # subnet, gateway, connected containers + their IPs
docker network rm my_net                # delete (must have no containers attached)
docker network prune                    # ➕ delete all unused networks

docker network connect my_net web       # ➕ attach a RUNNING container to a network
docker network disconnect my_net web    # ➕ detach it

docker inspect <container>              # full details, incl. the "Networks" section
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' web   # ➕ just the IP
```

**`docker network ls` on a fresh install** always shows these three:
```
NETWORK ID     NAME      DRIVER    SCOPE
a1b2c3d4e5f6   bridge    bridge    local
f6e5d4c3b2a1   host      host      local
0a9b8c7d6e5f   none      null      local
```

---

## 5. Lab: Two Containers Talking on a Bridge

```bash
# 1. Create a user-defined bridge network
docker network create my_bridge_network

# 2. Start two containers on it (sleep keeps alpine running, otherwise it exits at once)
docker run -d --network my_bridge_network --name container1 alpine sleep 1000
docker run -d --network my_bridge_network --name container2 alpine sleep 1000

# 3. Ping container2 BY NAME from container1
docker exec -it container1 ping -c 3 container2
# 64 bytes from 172.18.0.3: seq=0 ttl=64 time=0.09 ms   ✅ Docker's DNS turned the name into an IP

# 4. See both containers and their IPs
docker network inspect my_bridge_network
```

**➕ Prove the isolation:** a container on a different network can't reach them.
```bash
docker run -d --name outsider alpine sleep 1000            # joins the default bridge
docker exec -it outsider ping -c 2 container2              # ❌ bad address 'container2'
docker network connect my_bridge_network outsider          # add it to the network...
docker exec -it outsider ping -c 2 container2              # ✅ now it works
```

**➕ With `curl`** (the HTTP version of the same test):
```bash
docker run -d --network my_bridge_network --name web nginx
docker exec -it container1 wget -qO- http://web            # alpine has wget built in → nginx HTML
```
> No `-p` was needed. Containers on the same network reach each other's ports directly. **`-p` is only for traffic coming from outside the host.**

---

## 6. Lab: Isolation with the `none` Network

```bash
docker run -d --network none --name isolated_container alpine sleep 1000

docker exec -it isolated_container ping -c 2 google.com
# ping: bad address 'google.com'        ❌ no DNS, no network

docker exec -it isolated_container ip addr
# 1: lo: <LOOPBACK,UP>  inet 127.0.0.1/8   ← loopback only, no eth0
```

---

## 7. Docker Storage: Why Volumes?

A container's files live in its **thin writable layer** ([L5 §4](05-Docker.md#4-images-vs-containers)). That layer is **deleted with `docker rm`**.

```bash
docker run -d --name db -e POSTGRES_PASSWORD=x postgres
# ...app writes customer data...
docker rm -f db          # 💥 all the data is gone
```

**Volumes** store data **outside** the container, so it **survives** removal, upgrades and re-creation. They can also be **shared** between containers.

> 💡 **Simple words mein:** container = hotel ka kamra (checkout pe sab saaf). Volume = tumhara locker. Kamra badlo, locker ka saaman wahin rehta hai.

---

## 8. Named Volumes vs. Bind Mounts

```
   NAMED VOLUME                               BIND MOUNT
   Docker picks & manages the location        YOU pick an exact host folder
   /var/lib/docker/volumes/my_data/_data      /home/me/project/site-content
            │                                          │
            ▼                                          ▼
     container:/app                       container:/usr/share/nginx/html
```

| | Named volume | Bind mount |
|-|--------------|------------|
| Where data lives | Docker-managed folder (`/var/lib/docker/volumes/...`) | **Any path you choose** on the host |
| Created by | `docker volume create`, or **automatically** on first use | Folder must already exist |
| Managed with `docker volume` commands | ✅ | ❌ |
| Host edits show up in the container instantly | (not meant for that) | ✅ **Yes, live** |
| Works the same on every machine | ✅ Portable | ❌ Depends on the host's folder layout |
| **Best for** | **Databases, persistent app data (production)** | **Development:** live code reload, config files |

**➕ Third type, `tmpfs`:** data kept **only in RAM** and gone when the container stops. Good for secrets and temp files: `--mount type=tmpfs,target=/tmp`.

### `-v` vs. `--mount` (two ways to write the same thing)
```bash
# Named volume
docker run -v my_data:/app alpine ...
docker run --mount source=my_data,target=/app alpine ...

# Bind mount (a path instead of a name)
docker run -v "$(pwd)"/site-content:/usr/share/nginx/html nginx
docker run --mount type=bind,source="$(pwd)"/site-content,target=/usr/share/nginx/html nginx

# ➕ Read-only (container can read but not change)
docker run -v "$(pwd)"/config:/etc/app:ro myapp
```

| | `-v` | `--mount` |
|-|------|-----------|
| Style | Short, `src:dest[:ro]` | Long, `key=value` pairs, clearer |
| Bind-mount folder missing on host | **Silently creates** an empty folder | **Errors**, which catches typos ✅ |

> Docker's docs recommend `--mount` for clarity. `-v` is what you'll see most in tutorials.

---

## 9. Volume Commands

```bash
docker volume create my_data       # create
docker volume ls                   # list
docker volume inspect my_data      # details incl. "Mountpoint" (where it lives on the host)
docker volume rm my_data           # delete (fails if a container is using it)
docker volume prune                # ➕ delete all unused volumes ⚠️ data loss
```

```bash
$ docker volume inspect my_data
[{
    "Driver": "local",
    "Mountpoint": "/var/lib/docker/volumes/my_data/_data",
    "Name": "my_data",
    "Scope": "local"
}]
```

> ⚠️ `docker rm` does **not** delete named volumes. That's the whole point. Clean them up yourself, or use `docker rm -v` / `docker compose down -v` when you really want the data gone.

---

## 10. Lab: Data That Survives a Container

```bash
# 1. Create a volume
docker volume create my_data

# 2. Run a container with it mounted at /app (sleep keeps it running)
docker run -d --name my_container --mount source=my_data,target=/app alpine sleep 1000

# 3. Write a file into the volume
docker exec -it my_container sh -c 'echo "hello from container 1" > /app/note.txt'
docker exec -it my_container ls /app            # note.txt

# 4. Destroy the container completely
docker rm -f my_container

# 5. New container, same volume → the data is still there ✅
docker run --rm --mount source=my_data,target=/app alpine cat /app/note.txt
# hello from container 1
```

**➕ The real-world version: a database that survives restarts**
```bash
docker run -d --name pg -e POSTGRES_PASSWORD=secret \
  -v pgdata:/var/lib/postgresql/data postgres:16
# docker rm -f pg  →  run the same command again  →  all tables still there
```

---

## 11. Lab: Live-Editing a Website with a Bind Mount

```bash
# 1. Make a folder with a page on the HOST
mkdir site-content
echo "<h1>Version 1</h1>" > site-content/index.html

# 2. Mount it into nginx's web root
docker run -d --name webapp \
  --mount type=bind,source="$(pwd)"/site-content,target=/usr/share/nginx/html \
  -p 8080:80 nginx

curl localhost:8080          # <h1>Version 1</h1>

# 3. Edit on the HOST, with no rebuild and no restart
echo "<h1>Version 2 – edited live!</h1>" > site-content/index.html
curl localhost:8080          # <h1>Version 2 – edited live!</h1> ✅
```

> This is why developers use bind mounts: **edit code in VS Code, the container sees it instantly.** In production you'd **bake the files into the image** (or use a named volume) instead, so the server doesn't depend on a host folder.

**➕ In Compose:**
```yaml
services:
  web:
    image: nginx
    ports: ["8080:80"]
    volumes:
      - ./site-content:/usr/share/nginx/html   # bind mount (starts with ./ or /)
  db:
    image: postgres:16
    volumes:
      - pgdata:/var/lib/postgresql/data        # named volume
volumes:
  pgdata:
```

---

## 12. Quick Revision Cheat Sheet

```
DAEMON        dockerd manages images, containers, networks, volumes · no daemon = no Docker
NETWORKS      bridge (default, one host, NAT)   host (share host stack, -p ignored)
              none (loopback only, isolated)    overlay (many hosts, needs Swarm)
BRIDGE TIP    default bridge = IP only · user-defined bridge = DNS by container NAME ✅
              same network → talk directly (no -p) · -p only for outside traffic
NET CMDS      docker network create | ls | inspect | rm | prune | connect | disconnect
              docker inspect <container>  → "Networks" section, IP address
WHY VOLUMES   writable layer dies with docker rm → keep data OUTSIDE the container
NAMED VOL     Docker-managed (/var/lib/docker/volumes) · portable · databases / prod
BIND MOUNT    exact host path · live edits · dev / config · not portable
TMPFS ➕      RAM only, gone on stop
SYNTAX        -v my_data:/app          --mount source=my_data,target=/app
              -v $(pwd)/dir:/path      --mount type=bind,source=$(pwd)/dir,target=/path    :ro read-only
VOL CMDS      docker volume create | ls | inspect (Mountpoint) | rm | prune ⚠️
REMEMBER      docker rm keeps named volumes · rm -v / compose down -v delete them
```

---

## 13. Likely Exam / Interview Questions

1. **What is the Docker daemon and why is it important?** `dockerd` is the background service that manages images, containers, networks and volumes. Without it, no Docker command works.
2. **Name and explain Docker's four network types.** Bridge (default, single host), host (shares the host's network stack), none (isolated, loopback only), overlay (spans multiple hosts, used with Swarm).
3. **Which network does a container use by default?** The default `bridge` network.
4. **Why create your own bridge instead of using the default one?** A user-defined bridge has built-in DNS, so containers reach each other by name. It also isolates your app's containers from unrelated ones.
5. **Why does `-p` have no effect with `--network host`?** The container already uses the host's ports directly, so there's nothing to map.
6. **When would you use the `none` network?** For jobs that must be fully isolated: sensitive data processing or untrusted code.
7. **What problem does an overlay network solve?** It lets containers on different Docker hosts communicate as if they were on one network.
8. **Two containers on the same network: does one need `-p` to reach the other?** No. `-p` only exposes a port to traffic from outside the host.
9. **How do you find a container's IP address?** `docker inspect <container>` (the Networks section), or `docker network inspect <network>`.
10. **Why do we need volumes?** A container's writable layer is deleted with the container. Volumes keep data beyond the container's lifecycle and can be shared.
11. **Named volume vs. bind mount?** A named volume is managed by Docker and portable, best for databases and production. A bind mount maps an exact host folder with live changes, best for development.
12. **Where does Docker store named volumes on Linux?** `/var/lib/docker/volumes/<name>/_data` (shown as `Mountpoint` by `docker volume inspect`).
13. **`-v` vs. `--mount`?** Same feature, different syntax. `--mount` is more explicit and errors if a bind-mount source path is missing. `-v` silently creates it.
14. **Does `docker rm` delete a container's named volume?** No. Use `docker volume rm`, `docker rm -v`, or `docker compose down -v`.
15. **You changed a file on the host but the container still shows the old version. Why?** The files were probably `COPY`'d into the image, not bind-mounted. Rebuild the image, or use a bind mount during development.
