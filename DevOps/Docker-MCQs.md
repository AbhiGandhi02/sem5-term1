# Docker: Advanced MCQs

> **Practice for [Lecture 5 – Docker](05-Docker.md), [Lecture 6 – Lifecycle & Multi-Stage Builds](06-Docker-Lifecycle-and-Multi-Stage.md) and [Lecture 7 – Networking & Volumes](07-Docker-Networking-and-Volumes.md).** Each answer is shown right below its question. Cover it if you want to test yourself first.

| Part | Lecture | Questions |
|------|---------|-----------|
| [A](#part-a-lecture-5--images-containers--dockerfiles) | 5 – Images, Containers & Dockerfiles | Q1–Q10 |
| [B](#part-b-lecture-6--lifecycle-cleanup--multi-stage-builds) | 6 – Lifecycle, Cleanup & Multi-Stage Builds | Q11–Q20 |
| [C](#part-c-lecture-7--networking--volumes) | 7 – Networking & Volumes | Q21–Q30 |

---

## Part A: Lecture 5 – Images, Containers & Dockerfiles

### Q1. Linux containers on a Mac
Docker Desktop on macOS quietly runs a small Linux virtual machine. Why?
- A) To make containers faster than on Linux
- B) Containers **share the host's kernel**, and most images need a **Linux** kernel, which macOS doesn't have
- C) Docker Desktop only works inside a VM for licensing reasons
- D) To give each container its own operating system

> ✅ **Answer: B.** Containers aren't VMs: there's no guest OS inside each one. They use the host kernel, so on Mac/Windows Docker needs a Linux kernel to share, which is that hidden VM. · 📖 [L5 §2](05-Docker.md#2-containers-vs-virtual-machines)

### Q2. A slow rebuild
```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY . .
RUN npm install
CMD ["node", "server.js"]
```
You change one line in `server.js` and rebuild. What happens?
- A) Only the last layer is rebuilt
- B) `COPY . .` changes, so **every layer after it**, including `npm install`, is rebuilt
- C) Nothing is rebuilt, because the cache is per file
- D) The whole image is rebuilt from `FROM`

> ✅ **Answer: B.** Once a layer changes, all later layers are rebuilt. Copy `package*.json` and run `npm install` **before** `COPY . .`, so code edits don't reinstall dependencies. · 📖 [L5 §6](05-Docker.md#6-layers--caching)

### Q3. Starting the server with `RUN`
A Dockerfile ends with `RUN node server.js` (instead of `CMD`). What happens on `docker build`?
- A) The image builds, and the server starts whenever a container runs
- B) The **build hangs**: `RUN` executes during the build, and the server never exits, so that step never finishes
- C) Docker turns `RUN` into `CMD` automatically
- D) Syntax error

> ✅ **Answer: B.** `RUN` = **build time**, once, with the result saved in a layer. `CMD` = **run time**, every time a container starts. Long-running processes belong in `CMD`. · 📖 [L5 §5](05-Docker.md#5-the-dockerfile)

### Q4. `ENTRYPOINT` + `CMD`
An image has `ENTRYPOINT ["python"]` and `CMD ["app.py"]`. You run `docker run myimg worker.py`. What runs?
- A) `python app.py`
- B) `python worker.py`
- C) `worker.py`
- D) `python app.py worker.py`

> ✅ **Answer: B.** Arguments after the image name **replace `CMD`**, and `ENTRYPOINT` stays. (It's the same rule as Kubernetes `args` in [L8](08-Kubernetes-Basics.md#12-hands-on-a-run-once-task-pod).) · 📖 [L5 §5](05-Docker.md#5-the-dockerfile)

### Q5. `EXPOSE` without `-p`
The Dockerfile has `EXPOSE 3000`. You run `docker run -d myapp` (no `-p`). Can your browser reach `localhost:3000`?
- A) Yes, because `EXPOSE` publishes the port
- B) No: `EXPOSE` is only **documentation**. You need `-p 3000:3000` to publish it.
- C) Yes, but only on Linux
- D) Only if the app listens on `127.0.0.1`

> ✅ **Answer: B.** Publishing a port is always a **run-time** choice (`-p`), never something the image does by itself. · 📖 [L5 §5](05-Docker.md#5-the-dockerfile), [§9](05-Docker.md#9-port-mapping--docker-networking)

### Q6. Which port do you open?
You run `docker run -d -p 8080:3000 myapp`, and the app listens on 3000 inside the container. Which URL works from your laptop?
- A) `http://localhost:3000`
- B) `http://localhost:8080`
- C) Both
- D) Neither: the ports must match

> ✅ **Answer: B.** The format is `-p HOST:CONTAINER`. The host's 8080 forwards to the container's 3000. · 📖 [L5 §9](05-Docker.md#9-port-mapping--docker-networking)

### Q7. The container that vanished
`docker run -d myapp` prints a container ID, but `docker ps` shows **nothing**. What's going on, and what do you run next?
- A) Docker failed to create it, so reinstall Docker
- B) The container started and then **exited** (its main process finished or crashed). Use `docker ps -a`, then `docker logs <id>`.
- C) `-d` hides containers from `docker ps`
- D) It's paused

> ✅ **Answer: B.** `docker ps` lists **running** containers only. `-a` shows stopped ones with their exit code, and `logs` tells you why. · 📖 [L5 §8](05-Docker.md#8-core-commands)

### Q8. Data inside the container
A Postgres container stores data in its own filesystem (no volume). You run `docker rm -f db`, then start a new one from the same image. Is the data there?
- A) Yes, because data is saved in the image
- B) No: the data lived in the old container's **writable layer**, which was deleted with it
- C) Yes, but only if it's the same container name
- D) Only on Linux hosts

> ✅ **Answer: B.** Images are read-only. Each container adds its own thin writable layer, which disappears on `rm`. Use a volume: `-v pgdata:/var/lib/postgresql/data`. · 📖 [L5 §4](05-Docker.md#4-images-vs-containers)

### Q9. Push denied
You built `myapp:1.0` and ran `docker login`. Then `docker push myapp:1.0` fails with `denied: requested access to the resource is denied`. What's the fix?
- A) Push with `sudo`
- B) Tag it with your Docker Hub username first: `docker tag myapp:1.0 <user>/myapp:1.0`, then push that
- C) Use `docker save` instead
- D) Rename it to `myapp:latest`

> ✅ **Answer: B.** Without a namespace, the name means the official library repo, which you can't push to. Your images live under `<username>/`. · 📖 [L5 §10](05-Docker.md#10-registries-docker-hub-ecr-acr)

### Q10. `localhost` in Compose
In a `docker-compose.yml`, the `web` service connects to Postgres using `DB_HOST=localhost`, and gets `Connection refused`. The `db` service is running. What's the fix?
- A) Add `network_mode: host` to `db`
- B) Use `DB_HOST=db`: each container's `localhost` is **itself**, and services reach each other by **service name**
- C) Publish port 5432 with `-p` on `db`
- D) Start `web` first

> ✅ **Answer: B.** Compose puts all services on one network with built-in DNS. `localhost` inside `web` means the `web` container. · 📖 [L5 §11](05-Docker.md#11-docker-compose-multi-container-apps)

---

## Part B: Lecture 6 – Lifecycle, Cleanup & Multi-Stage Builds

### Q11. Exit code 137
`docker ps -a` shows `Exited (137) 2 minutes ago`. What most likely happened?
- A) The app finished successfully
- B) The process was **killed with SIGKILL**, often for using too much memory (OOM)
- C) The image tag was wrong
- D) The container was paused

> ✅ **Answer: B.** 137 = 128 + 9 (SIGKILL). `0` = success, `1` = app error, `143` = SIGTERM (a normal `docker stop`). · 📖 [L6 §3](06-Docker-Lifecycle-and-Multi-Stage.md#3-the-full-container-lifecycle)

### Q12. Pause vs. stop
What's the key difference between `docker pause web` and `docker stop web`?
- A) None
- B) `pause` **freezes** the processes (memory kept, resumes instantly with `unpause`). `stop` sends SIGTERM, the process **ends**, and the container becomes Exited.
- C) `pause` deletes the container
- D) `stop` keeps memory, `pause` frees it

> ✅ **Answer: B.** Pause = the video paused at that frame. Stop = the program is closed and restarts from the beginning on `docker start`. · 📖 [L6 §3](06-Docker-Lifecycle-and-Multi-Stage.md#3-the-full-container-lifecycle)

### Q13. Removing a running container
`docker rm web` fails with `You cannot remove a running container`. What are your two options?
- A) `docker kill` the daemon, or reboot
- B) `docker stop web` then `docker rm web`, **or** `docker rm -f web`
- C) `docker rmi web`
- D) `docker pause web` then `docker rm web`

> ✅ **Answer: B.** The daemon protects running containers from accidental deletion. `-f` says "yes, I mean it" (it stops and removes). A paused container is still running, so D fails. · 📖 [L6 §6](06-Docker-Lifecycle-and-Multi-Stage.md#6-common-errors--fixes)

### Q14. Image in use
`docker rmi nginx` fails with `image is being used by stopped container 3f2a…`. Why does a **stopped** container block it?
- A) It doesn't: the error is a bug
- B) A stopped container still exists and depends on that image's layers. Remove the container first (`docker rm 3f2a`), or force it with `rmi -f`.
- C) Stopped containers keep running in the background
- D) You must `docker logout` first

> ✅ **Answer: B.** Stopped ≠ removed. The container's writable layer sits on top of the image's layers. · 📖 [L6 §6](06-Docker-Lifecycle-and-Multi-Stage.md#6-common-errors--fixes)

### Q15. The dangerous prune
Which command can delete a stopped database container's **data** if you aren't careful?
- A) `docker system df`
- B) `docker system prune --volumes`
- C) `docker image prune`
- D) `docker builder prune`

> ✅ **Answer: B.** `--volumes` also removes volumes not used by a container, including your DB data if its container was removed. Always check `docker system df` first. · 📖 [L6 §7](06-Docker-Lifecycle-and-Multi-Stage.md#7-managing-disk-space-system-df--prune)

### Q16. `<none>:<none>` images
After rebuilding `myapp:latest` several times, `docker images` shows several `<none>  <none>` entries. What are they?
- A) Corrupted images
- B) **Dangling** images: the old builds whose tag moved to the newer image. Safe to remove with `docker image prune`.
- C) Images that failed to build
- D) Images from other users

> ✅ **Answer: B.** Each rebuild moves the `myapp:latest` tag to the new image, and the old one loses its name. · 📖 [L6 §7](06-Docker-Lifecycle-and-Multi-Stage.md#7-managing-disk-space-system-df--prune)

### Q17. What's in a multi-stage image?
```dockerfile
FROM node:20 AS builder
WORKDIR /app
COPY . .
RUN npm install && npm run build
FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
```
What does the final image contain?
- A) Node, `node_modules`, the source code and nginx
- B) Only `nginx:alpine` plus the built `dist/` files. Everything else in the builder stage is thrown away.
- C) Both stages merged together
- D) Only the builder stage

> ✅ **Answer: B.** Only the **last** stage becomes the image. `COPY --from=builder` pulls just the files you name out of the earlier stage. · 📖 [L6 §8](06-Docker-Lifecycle-and-Multi-Stage.md#8-multi-stage-builds)

### Q18. Why multi-stage helps security
Besides a smaller size, why are multi-stage images considered **more secure**?
- A) They're encrypted
- B) Compilers, package managers and dev dependencies aren't in the production image, so there are fewer vulnerable packages and fewer tools for an attacker to use
- C) They always run as root
- D) Docker Hub scans them for free

> ✅ **Answer: B.** Less software means a smaller attack surface and fewer CVEs for scanners like Trivy to find. · 📖 [L6 §8](06-Docker-Lifecycle-and-Multi-Stage.md#8-multi-stage-builds), [§9](06-Docker-Lifecycle-and-Multi-Stage.md#9-docker--devsecops-secure-images)

### Q19. A secret in the Dockerfile
A Dockerfile has `ENV API_KEY=sk-live-123`. The image is pushed to a private registry. Who can read the key?
- A) Nobody: `ENV` values are encrypted
- B) **Anyone who can pull the image**, using `docker inspect` or `docker history`
- C) Only the image author
- D) Only containers started with `-e API_KEY`

> ✅ **Answer: B.** Everything in a layer stays in the image's history. Pass secrets at **run time** (`-e`, or Kubernetes Secrets), never bake them in. · 📖 [L6 §9](06-Docker-Lifecycle-and-Multi-Stage.md#9-docker--devsecops-secure-images)

### Q20. A fix made with `exec`
You ran `docker exec -it web sh` and installed `curl` with `apk add curl`. Later the container is removed and a new one is started from the same image. Is `curl` there?
- A) Yes, because `exec` changes the image
- B) No: the change lived in the old container's writable layer. Add `RUN apk add curl` to the Dockerfile and rebuild.
- C) Yes, if you used `-it`
- D) Only if the container name is the same

> ✅ **Answer: B.** `exec` is for **looking around and debugging**. Permanent changes go in the Dockerfile. · 📖 [L6 §5](06-Docker-Lifecycle-and-Multi-Stage.md#5-looking-inside-docker-exec)

---

## Part C: Lecture 7 – Networking & Volumes

### Q21. Name lookup on the default bridge
Two containers were started **without** `--network`. From `c1`, `ping c2` fails with `bad address 'c2'`, but pinging `c2`'s IP works. Why?
- A) ICMP is blocked on bridges
- B) The **default** `bridge` network has **no DNS** for container names. A user-defined network (`docker network create`) does.
- C) Container names must be uppercase
- D) `c2` must publish a port first

> ✅ **Answer: B.** Always create your own network for containers that talk to each other. Compose does this for you. · 📖 [L7 §3.1](07-Docker-Networking-and-Volumes.md#31-bridge-default)

### Q22. `-p` with host networking
You run `docker run -d --network host -p 8080:80 nginx` on a Linux server. Where is nginx reachable?
- A) `<server-ip>:8080`
- B) `<server-ip>:80`: with `host` networking the container uses the host's ports directly, and `-p` is **ignored**
- C) Both 80 and 8080
- D) Nowhere: the two flags conflict

> ✅ **Answer: B.** No separate network stack means nothing to map. If something else already uses port 80 on the host, nginx fails to start. · 📖 [L7 §3.2](07-Docker-Networking-and-Volumes.md#32-host)

### Q23. The `none` network
A container runs with `--network none`. What does `ip addr` inside it show?
- A) An `eth0` with a `172.17.x.x` address
- B) Only the **loopback** interface (`lo`, `127.0.0.1`)
- C) The host's interfaces
- D) Nothing at all

> ✅ **Answer: B.** No `eth0`, so no other containers and no internet. Used for fully isolated jobs. · 📖 [L7 §3.3](07-Docker-Networking-and-Volumes.md#33-none)

### Q24. Across two servers
Containers on **server A** and **server B** must talk as if they're on one network. Which driver, and what does it need?
- A) `bridge`, nothing extra
- B) `overlay`, which needs Docker **Swarm** (or another orchestrator) to coordinate the hosts
- C) `host`, plus port forwarding
- D) `none`

> ✅ **Answer: B.** Bridge, host and none are all **single-host**. Overlay spans hosts. (Kubernetes solves the same problem with its own networking.) · 📖 [L7 §3.4](07-Docker-Networking-and-Volumes.md#34-overlay)

### Q25. Do containers need `-p` between them?
`api` and `db` are on the same user-defined network. `db` was started **without** `-p`. Can `api` connect to `db:5432`?
- A) No, because the port isn't published
- B) Yes: containers on the same network reach each other's ports directly. `-p` is only for traffic coming from **outside the host**.
- C) Only if `api` uses `--network host`
- D) Only over UDP

> ✅ **Answer: B.** Leaving `-p` off the database is good practice: it isn't exposed to the outside world at all. · 📖 [L7 §5](07-Docker-Networking-and-Volumes.md#5-lab-two-containers-talking-on-a-bridge)

### Q26. Live editing during development
While developing, you want edits in VS Code to appear inside the container **immediately**, without rebuilding. What do you use?
- A) A named volume
- B) A **bind mount** of your project folder (`-v "$(pwd)":/app`)
- C) `COPY . .` in the Dockerfile
- D) `tmpfs`

> ✅ **Answer: B.** A bind mount maps an exact host folder, so changes show up live. Named volumes are Docker-managed storage (best for DB data), and `COPY` freezes files at build time. · 📖 [L7 §8](07-Docker-Networking-and-Volumes.md#8-named-volumes-vs-bind-mounts)

### Q27. A typo in the bind-mount path
You mistype the host path: `-v "$(pwd)"/sitecontent:/usr/share/nginx/html` (the real folder is `site-content`). What happens?
- A) Docker errors out: path not found
- B) Docker **silently creates** an empty `sitecontent` folder and mounts it, so nginx serves an empty directory
- C) Docker uses `site-content` automatically
- D) The container mounts nothing

> ✅ **Answer: B.** `-v` creates missing bind-mount sources. `--mount type=bind,...` would **error** instead, which catches typos. That's why `--mount` is recommended. · 📖 [L7 §8](07-Docker-Networking-and-Volumes.md#8-named-volumes-vs-bind-mounts)

### Q28. Does `docker rm` delete the volume?
A container used the named volume `pgdata`. You run `docker rm -f db`. What happens to `pgdata`?
- A) It's deleted with the container
- B) It **stays**. Named volumes outlive containers, which is the point. Remove it explicitly with `docker volume rm pgdata` (or `docker compose down -v`).
- C) It's converted to a bind mount
- D) It's moved to `/tmp`

> ✅ **Answer: B.** Persistence is exactly why volumes exist. Clean up deliberately, or with `docker volume prune`, carefully. · 📖 [L7 §9](07-Docker-Networking-and-Volumes.md#9-volume-commands)

### Q29. A volume in use
`docker volume rm pgdata` fails with `volume is in use`. What's the reason?
- A) Volumes can never be removed
- B) A container (running **or stopped**) still references it. Remove that container first.
- C) You need `sudo`
- D) The volume is a bind mount

> ✅ **Answer: B.** Like images (Q14), a stopped container still counts as a user of the volume. · 📖 [L7 §9](07-Docker-Networking-and-Volumes.md#9-volume-commands)

### Q30. Where named volumes live
On a Linux host, where does Docker keep the data of the named volume `my_data`?
- A) In the image
- B) `/var/lib/docker/volumes/my_data/_data` (shown as `Mountpoint` by `docker volume inspect`)
- C) In the current folder
- D) In `/tmp/my_data`

> ✅ **Answer: B.** Docker manages that location. ➕ On Docker Desktop (Mac/Windows), the path is inside the hidden Linux VM, not on your laptop's disk. · 📖 [L7 §9](07-Docker-Networking-and-Volumes.md#9-volume-commands)
