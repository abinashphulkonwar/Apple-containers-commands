# Apple `container` CLI — Documentation

`container` is Apple's native, open-source command-line tool for creating and running **Linux containers** on macOS. Instead of a shared Linux VM (the Docker Desktop model), each container gets its own lightweight, sub-second-boot virtual machine, built on Apple's `Containerization` framework and optimized for Apple silicon.

- **Requires:** Apple silicon Mac (M1 or later), macOS 26 for the full feature set (network/DNS commands need macOS 26+; basic use works on macOS 15+)
- **Written in:** Swift
- **Install:** signed `.pkg` from the [apple/container GitHub releases](https://github.com/apple/container/releases)
- **Source:** https://github.com/apple/container

> Command availability can vary slightly by macOS/tool version. Run `container <command> --help` any time to confirm flags for your installed version.

---

## Table of Contents

1. [System Commands](#1-system-commands)
2. [Core Commands (run & build)](#2-core-commands-run--build)
3. [Container Lifecycle Management](#3-container-lifecycle-management)
4. [Image Management](#4-image-management)
5. [Builder (BuildKit) Management](#5-builder-buildkit-management)
6. [Network Management](#6-network-management-macos-26)
7. [Volume Management](#7-volume-management)
8. [Registry Management](#8-registry-management)
9. [Container Machine Management](#9-container-machine-management)
10. [Kubernetes Cluster Management (experimental)](#10-kubernetes-cluster-management-experimental)
11. [Your Original Commands — Annotated](#11-your-original-commands--annotated)
12. [Handy Workflows](#12-handy-workflows)

---

## 1. System Commands

These control the background services (`container-apiserver`) that everything else depends on. Nothing else works until the system is started.

### `container system start`
Starts the container services (`container-apiserver` and friends) via launchd. On first run it will prompt to install a default Linux kernel.

```bash
container system start

# start without being asked about kernel install
container system start --enable-kernel-install

# custom data/log locations, with a startup timeout
container system start --app-root ~/containers --log-root ~/containers/logs --timeout 30
```

### `container system stop`
Stops the container services and de-registers them from launchd.

```bash
container system stop

# if services were started with a custom launchd prefix
container system stop --prefix com.mycompany.container.
```

### `container system status`
Health-checks the API server and prints whether the system is running.

```bash
container system status
container system status --format json
```

### `container system version`
Prints CLI and (if reachable) API server version info.

```bash
container system version
```

### `container system logs`
Tails or fetches logs for the container services themselves (not container app logs — see `container logs` for that).

```bash
container system logs --follow
container system logs --last 30m
```

### `container system df`
Shows disk usage summary across images, containers, and volumes (counts, sizes, reclaimable space) — similar to `docker system df`.

```bash
container system df
```

### `container system dns create` / `delete` / `list`
Manages a **local DNS domain** so containers can be reached by name from the host (e.g., `myapp.test`). Requires `sudo`.

```bash
sudo container system dns create test
sudo container system dns delete test
container system dns list
```

### `container system kernel set`
Installs/updates the Linux kernel the runtime boots containers with.

```bash
# install Apple's recommended default kernel
container system kernel set --recommended

# install a specific kernel binary
container system kernel set --binary ./vmlinux --arch arm64 --force
```

### `container system property list`
Lists current system configuration properties (TOML by default).

```bash
container system property list
container system property list --format json
```

---

## 2. Core Commands (run & build)

### `container run`
The single most-used command — pulls (if needed) and runs a container from an image.

```bash
container run [<options>] <image> [<arguments> ...]
```

Key flags (grouped):

| Category | Flag | Meaning |
|---|---|---|
| Process | `-i, --interactive` | Keep stdin open |
| Process | `-t, --tty` | Allocate a TTY |
| Process | `-e, --env key=val` | Set an env var |
| Process | `--env-file <file>` | Load env vars from a file |
| Process | `-w, --workdir <dir>` | Initial working directory |
| Process | `-u, --user`, `--uid`, `--gid` | Run as a specific user/uid/gid |
| Resource | `-c, --cpus <n>` | vCPUs for this container |
| Resource | `-m, --memory <size>` | Memory limit, e.g. `1G` |
| Management | `-d, --detach` | Run in background |
| Management | `--name <name>` | Assign a container name |
| Management | `--rm, --remove` | Auto-delete when it stops |
| Management | `-p, --publish host:container` | Publish a port |
| Management | `-v, --volume src:dst` | Bind-mount a volume/host path |
| Management | `--network <name>` | Attach to a user-defined network |
| Management | `--entrypoint <cmd>` | Override image entrypoint |
| Management | `--init` | Run an init process (signal forwarding, zombie reaping) |
| Management | `--cap-add` / `--cap-drop` | Add/drop Linux capabilities |
| Management | `-a, --arch`, `--os`, `--platform` | Target architecture/OS for multi-platform images |
| Management | `--rosetta` | Enable Rosetta translation inside the VM |
| Management | `--dns`, `--dns-domain`, `--dns-search` | Configure DNS in-container |

```bash
# interactive shell
container run -it ubuntu:latest /bin/bash

# background web server with a published port
container run -d --name web -p 8080:80 nginx:latest

# constrained resources + env vars
container run -e NODE_ENV=production --cpus 2 --memory 1G node:18

# ephemeral dev container with your project mounted in, node_modules kept in a named volume
container run --rm -p 3000:3000 -v $(pwd):/app -v app_node_modules:/app/node_modules imagename
```

### `container build`
Builds an OCI image from a Dockerfile/Containerfile using an isolated BuildKit builder VM.

```bash
container build [<options>] [<context-dir>]
```

| Flag | Meaning |
|---|---|
| `-t, --tag <name>` | Tag for the built image (repeatable) |
| `-f, --file <path>` | Path to Dockerfile (defaults to `Dockerfile`, then `Containerfile`) |
| `--build-arg key=val` | Build-time variable |
| `--target <stage>` | Build only up to a specific multi-stage target |
| `--no-cache` | Disable layer caching |
| `--pull` | Always pull the latest base image |
| `-c, --cpus`, `-m, --memory` | Resources for the *builder* |
| `--secret id=...` | Pass build-time secrets |
| `--ssh default` | Forward SSH agent to the build |
| `-o, --output type=oci\|tar\|local` | Where/how to emit the build output |
| `--platform os/arch` | Target platform for the image |

```bash
# basic build from a dev Dockerfile
container build -t imagename -f Dockerfile.dev .

# multi-tag production build, no cache
container build --target production --no-cache -t my-app:prod -t my-app:latest .
```

---

## 3. Container Lifecycle Management

| Command | Purpose |
|---|---|
| `container create` | Create a container from an image **without starting it** (same flags as `run`) |
| `container start <id>` | Start a previously-created/stopped container; `-a` attach output, `-i` attach stdin |
| `container stop [<ids>] [--all]` | Gracefully stop (SIGTERM, then SIGKILL after `--time`, default 5s) |
| `container kill [<ids>] [--all]` | Immediately send a signal (default `KILL`) — no graceful shutdown |
| `container delete (rm) [<ids>] [--all] [--force]` | Delete containers; `--force` to remove running ones |
| `container list (ls) [--all]` | List containers (running only, unless `--all`) |
| `container exec <id> <cmd>` | Run an extra command inside a running container |
| `container logs <id> [--follow] [-n N] [--boot]` | View stdout/stderr or boot logs |
| `container inspect <ids>` | Full JSON metadata for one or more containers |
| `container stats [<ids>] [--no-stream]` | Live (or one-shot) CPU/mem/network/I/O usage, like `top` |
| `container copy (cp) <src> <dst>` | Copy files between host and a running container (`id:/path` syntax) |
| `container export <id> [-o file.tar]` | Export a container's filesystem as a tar archive |
| `container clean <ids>` | Clean unused space inside a *running* container's root fs and volume mounts |
| `container prune` | Delete all stopped containers to reclaim space |

```bash
# create then start later
container create --name db postgres:16
container start -a -i db

# stop everything gracefully, force-kill after 10s
container stop --all --time 10

# tail logs live
container logs -f web

# live resource dashboard
container stats

# copy a file out of a container
container cp web:/var/log/app.log ./app.log

# remove all stopped containers
container prune
```

---

## 4. Image Management

| Command | Purpose |
|---|---|
| `container image list (ls) [--verbose]` | List local images |
| `container image pull <ref>` | Pull an image from a registry |
| `container image push <ref>` | Push an image to a registry |
| `container image save <refs> -o out.tar` | Save image(s) to a tar archive |
| `container image load --input in.tar` | Load image(s) from a tar archive |
| `container image tag <src> <target>` | Add a new tag to an existing image |
| `container image delete (rm) <images> [--all]` | Delete image(s) |
| `container image prune [--all]` | Remove dangling images (or all unused with `-a`) |
| `container image inspect <images>` | Full JSON metadata for image(s) |

```bash
# list images
container image ls

# pull a specific platform variant
container image pull --platform linux/amd64 redis:7

# remove a development image
container image rm blog-post-react

# save/load for offline transfer
container image save -o app.tar myapp:latest
container image load --input app.tar

# reclaim space from dangling images only
container image prune
# ...or every image not used by a container
container image prune --all
```

---

## 5. Builder (BuildKit) Management

`container build` runs inside a dedicated BuildKit builder container. These commands manage that builder directly.

| Command | Purpose |
|---|---|
| `container builder start [--cpus] [--memory]` | Start the BuildKit builder (with optional resource limits) |
| `container builder status` | Show whether the builder is running |
| `container builder stop` | Stop the builder |
| `container builder delete (rm) [--force]` | Delete the builder container |

```bash
container builder start --cpus 4 --memory 4G
container builder status
```

---

## 6. Network Management (macOS 26+)

User-defined networks let containers reach each other by name, similar to a Docker bridge network.

| Command | Purpose |
|---|---|
| `container network create <name> [--subnet CIDR] [--internal]` | Create a network |
| `container network list (ls)` | List networks |
| `container network inspect <names>` | Show network details |
| `container network delete (rm) <names> [--all]` | Delete network(s) |
| `container network prune` | Remove unused networks (defaults/system ones are preserved) |

```bash
container network create appnet --subnet 192.168.100.0/24
container run -d --network appnet --name api myapi:latest
container run -d --network appnet --name db postgres:16
# 'api' can now reach 'db' by container name
```

---

## 7. Volume Management

Named volumes persist data outside a container's lifecycle. They can be created explicitly, or implicitly the first time they're referenced in `-v name:/path`.

| Command | Purpose |
|---|---|
| `container volume create <name> [-s size] [--opt key=val]` | Create a named volume |
| `container volume list (ls)` | List volumes |
| `container volume inspect <names>` | Show volume details |
| `container volume delete (rm) <names> [--all]` | Delete volume(s) (must be unused) |
| `container volume prune` | Remove all volumes not referenced by any container |

```bash
# list all named volumes
container volume ls

# remove specific named volumes
container volume rm app_node_modules_bpr npm_global_cache

# create a volume with a size cap and ext4 journaling mode
container volume create --opt journal=ordered -s 10g build_cache

# remove every unused volume
container volume prune
```

> Anonymous volumes (created via `-v /path` with no source name) are tagged with the `com.apple.container.resource.anonymous` label so you can find and clean them up separately.

---

## 8. Registry Management

| Command | Purpose |
|---|---|
| `container registry login <server> [-u user] [--password-stdin]` | Authenticate to a registry |
| `container registry logout <server>` | Remove stored credentials |
| `container registry list` | List logged-in registries |

```bash
container registry login ghcr.io -u myuser --password-stdin < token.txt
container registry logout ghcr.io
```

---

## 9. Container Machine Management

A **container machine** (`container machine`, aliased `m`) is a longer-lived Linux VM you can log into directly — useful for workflows closer to a traditional VM than a single-purpose container.

| Command | Purpose |
|---|---|
| `container machine create <image> [--name] [--cpus] [--memory] [--no-boot]` | Create (and boot) a container machine |
| `container machine run [--name]` | Open a shell / run a command in a machine (boots it if needed) |
| `container machine list (ls)` | List machines |
| `container machine inspect [<id>]` | Show machine details |
| `container machine set <key=val>...` | Change CPU/memory/home-mount/kernel settings (applies on next boot) |
| `container machine set-default <id>` | Make a machine the default target |
| `container machine logs [<id>]` | View machine logs |
| `container machine stop [<id>]` | Stop a machine |
| `container machine delete (rm) <id>` | Delete a machine |

```bash
container machine create alpine:3.22 --name devbox --cpus 4 --memory 8G --set-default
container machine run -n devbox uname -a
container machine set -n devbox memory=4G
container machine stop devbox
```

---

## 10. Kubernetes Cluster Management (experimental)

`container k8s` spins up a local single-node Kubernetes cluster (via `kindest/node` + `kubeadm`) backed by a container VM — handy for local K8s testing without a separate tool like kind/minikube.

| Command | Purpose |
|---|---|
| `container k8s create [--name] [--node-image] [--cpus] [--memory] [--rm]` | Create and start a cluster; merges credentials into `~/.kube/config` |
| `container k8s start [--name]` | Start a stopped cluster and refresh its kubeconfig entry |
| `container k8s list (ls)` | List clusters and their status |
| `container k8s load-image <image> [--name] [--platform]` | Import a local image into the cluster's containerd so pods can use it |
| `container k8s write-config [--name] [--kubeconfig]` | (Re)write cluster credentials to a kubeconfig file |
| `container k8s delete (rm) [--name]` | Stop and delete a cluster |

```bash
container k8s create --name my-cluster --cpus 4 --memory 8g
container k8s load-image my-app:latest --name my-cluster
kubectl get nodes           # works via ~/.kube/config
container k8s delete --name my-cluster
```

---

## 11. Your Original Commands — Annotated

| Command | Explanation |
|---|---|
| `container system start` | Boots the container services (apiserver + supporting daemons). Must run before any other command. |
| `container system stop` | Shuts down those services and unregisters them from launchd. |
| `container volume ls` | Lists every named volume the tool knows about. |
| `container volume rm app_node_modules_bpr npm_global_cache` | Deletes the two named volumes `app_node_modules_bpr` and `npm_global_cache` (fails if still attached to a container). |
| `container image ls` | Lists all locally stored images. |
| `container image rm blog-post-react` | Deletes the local image tagged `blog-post-react`. |
| `container system prune` | Removes stopped containers, unused networks, and dangling (untagged) images — but **not** volumes. |
| `container system prune --volumes` | Same as above, but also removes unused volumes. |
| `container build -t imagename -f Dockerfile.dev .` | Builds an image named `imagename` using `Dockerfile.dev` as the build recipe, with the current directory (`.`) as the build context. |
| `container run --rm -p 3000:3000 -v $(pwd):/app -v app_node_modules:/app/node_modules imagename` | Runs `imagename`, auto-removing the container on exit (`--rm`), mapping host port 3000 to container port 3000, bind-mounting the current directory into `/app`, and using the named volume `app_node_modules` for `/app/node_modules` — a classic pattern so your host bind-mount doesn't clobber container-installed `node_modules`. *(Note: your original snippet was missing the `-v` flag before `app_node_modules:/app/node_modules` — added above.)* |

> ⚠️ Apple's `system prune` today targets containers/networks/images. Volumes are pruned separately via `--volumes` on `system prune`, or directly with `container volume prune`.

---

## 12. Handy Workflows

**Full local dev reset:**
```bash
container stop --all
container system prune --volumes
container image prune --all
```

**Build → run → tail logs:**
```bash
container build -t myapp:dev -f Dockerfile.dev .
container run -d --name myapp -p 3000:3000 myapp:dev
container logs -f myapp
```

**Two containers talking over a private network:**
```bash
container network create appnet
container run -d --network appnet --name db postgres:16
container run -d --network appnet -p 3000:3000 --name api myapp:dev
```

**Inspect resource usage while developing:**
```bash
container stats myapp
```

**Ship an image without a registry:**
```bash
container image save -o myapp.tar myapp:dev
# ...transfer myapp.tar to another Mac...
container image load --input myapp.tar
```

---

### Reference
Official docs: https://github.com/apple/container/blob/main/docs/command-reference.md
