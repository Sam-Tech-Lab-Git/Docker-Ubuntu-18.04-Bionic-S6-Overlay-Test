# Docker Ubuntu 18.04 LTS (Bionic) + s6-overlay - Sam Tech Lab

<table align="center">
  <tr>
    <td align="center" width="50%">
      <a href="https://github.com/Sam-Tech-Lab-Git" target="_blank">
        <img src="https://raw.githubusercontent.com/Sam-Dz-Devops/Images/main/Sam-Tech-Site-Web.png"
             alt="Sam Tech Lab Logo" width="300"/>
      </a>
    </td>
    <td align="center" width="50%">
      <a href="https://github.com/Sam-Tech-Lab-Git/Docker-Ubuntu-18.04-Bionic-S6-Overlay-Test" target="_blank">
        <img src="https://upload.wikimedia.org/wikipedia/commons/7/76/Ubuntu-logo-2022.svg?sanitize=true"
             alt="Ubuntu Logo" width="180"/>
      </a>
    </td>
  </tr>
</table>

<p align="center">
  <a href="https://github.com/Sam-Tech-Lab-Git" target="_blank">
    <img src="https://img.shields.io/static/v1?label=SamTechLab&message=GitHub&color=94398d&labelColor=555555&style=for-the-badge&logo=github&logoColor=white" alt="GitHub"/>
  </a>
  <a href="https://github.com/Sam-Tech-Lab-Git/Docker-Ubuntu-18.04-Bionic-S6-Overlay-Test/blob/main/LICENSE" target="_blank">
    <img src="https://img.shields.io/badge/License-MIT-blue.svg?style=for-the-badge" alt="License: MIT"/>
  </a>
  <a href="https://github.com/just-containers/s6-overlay" target="_blank">
    <img src="https://img.shields.io/badge/s6--overlay-3.2.3.2-brightgreen.svg?style=for-the-badge" alt="s6-overlay 3.2.3.2"/>
  </a>
  <a href="https://github.com/Sam-Tech-Lab-Git/Docker-Ubuntu-18.04-Bionic-S6-Overlay-Test/actions/workflows/build-multi-arch.yml" target="_blank">
      <img src="https://github.com/Sam-Tech-Lab-Git/Docker-Ubuntu-18.04-Bionic-S6-Overlay-Test/actions/workflows/build-multi-arch.yml/badge.svg" alt="Build multi-arch"/>
  </a>
  <a href="https://github.com/Sam-Tech-Lab-Git/Docker-Ubuntu-18.04-Bionic-S6-Overlay-Test/actions/workflows/vuln-scan.yml" target="_blank">
      <img src="https://github.com/Sam-Tech-Lab-Git/Docker-Ubuntu-18.04-Bionic-S6-Overlay-Test/actions/workflows/vuln-scan.yml/badge.svg" alt="Vulnerability Scan"/>
  </a>
</p>

<p align="center">
  <b>English</b> · <a href="#documentation-française">Version française ↓</a>
</p>

---

## Quickstart

```bash
# Run a shell (as the unprivileged appuser)
docker run -it --rm ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic-s6:latest

# Match the container user to your host user
docker run --rm -e PUID=$(id -u) -e PGID=$(id -g) \
  -v "$PWD/data:/config" \
  ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic-s6:latest id appuser
```

Build on top of it:

```dockerfile
FROM ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic-s6:latest

RUN apt-get update && \
    apt-get install -y --no-install-recommends your-package && \
    rm -rf /var/lib/apt/lists/*

COPY root/ /
```

**Contents:** [Overview](#overview) · [How the container boots](#how-the-container-boots) ·
[Image reference](#image-reference) · [Getting started](#getting-started) ·
[Adding your own services](#adding-your-own-services) ·
[Running several services](#running-several-services) · [Logging](#logging) ·
[Complete example NGINX](#complete-example-nginx) · [Security model](#security-model) ·
[Troubleshooting](#troubleshooting) · [Maintenance](#maintenance)

---

## Overview

A **minimal, hardened, multi-architecture Ubuntu 18.04 LTS (Bionic) base image**, built
`FROM scratch` from the **official Ubuntu OCI rootfs**, with
[s6-overlay](https://github.com/just-containers/s6-overlay) as its init system and process
supervisor.

It is designed as a **foundation for your own images**: it ships an init system, a non-root user
with runtime-configurable UID/GID, and a hardened system baseline — then gets out of your way.

> **Supported architectures:** `linux/amd64`, `linux/arm64`
> **Automatic monthly rebuilds** pick up whatever Ubuntu still publishes for Bionic — read
> [Base image support status](#base-image-support-status) before deploying this in production.

### Key features

- ✅ **Built `FROM scratch`** from the official Ubuntu OCI rootfs — no third-party base layer
- ✅ **s6-overlay as PID 1** — zombie reaping, ordered startup and shutdown, correct signal handling
- ✅ **Runtime-configurable `PUID` / `PGID`** — applied at container start, before any service runs
- ✅ **Multi-service supervision** with declared dependencies and automatic restart
- ✅ **Per-service log rotation** built in, via `logutil-service`
- ✅ **Non-root by default** — the default `CMD` drops privileges to `appuser`
- ✅ **System hardening** — `root` account locked, SUID/SGID bits stripped, world-writable bits
  removed, `umask 027`
- ✅ **Supply-chain integrity** — Alpine builder pinned by digest, s6-overlay tarballs pinned by
  SHA256 and verified before extraction, CI actions pinned by commit SHA
- ✅ **Fail-fast init** — a failing init script stops the container instead of running on with a
  broken state
- ✅ **APT & dpkg optimisation** — no recommended/suggested packages, no translations, clean cache
- ✅ **Continuously verified** — hadolint, shellcheck, 9 container integration tests run on
  **both architectures**, and weekly Trivy scans

---

## How the container boots

Understanding the boot sequence explains where your own code hooks in:

```
docker run
   │
   ├─ 1. ENTRYPOINT /init            s6-overlay takes PID 1
   │
   ├─ 2. s6-rc oneshots              init-adduser applies PUID/PGID
   │        (dependency-ordered)     ← your init tasks go here
   │
   ├─ 3. s6-rc longruns              supervised daemons start
   │        (dependency-ordered)     ← your services go here
   │
   └─ 4. CMD                         runs as a normal process
                                     container exits when it exits
```

On shutdown (`docker stop`, or the `CMD` exiting), the sequence runs in reverse: services are
stopped in dependency order, then remaining processes get `SIGTERM`, then `SIGKILL` after a grace
period.

---

## Image reference

### Registries and tags

| Registry | Image | Architectures |
|---|---|---|
| GHCR | `ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic-s6:latest` | amd64 + arm64 |
| GHCR | `ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic-s6:YYYY.MM` | amd64 + arm64 |
| Docker Hub | `samtechlab/ubuntu-18.04-bionic-s6:latest` | amd64 + arm64 |
| Docker Hub | `samtechlab/ubuntu-18.04-bionic-s6:YYYY.MM` | amd64 + arm64 |

Tags point at a multi-architecture manifest — Docker automatically selects the right image for
the host platform. `latest` tracks the monthly rebuild.

**Neither tag is immutable.** A `YYYY.MM` tag names the month a build ran, not one specific build:
any build during that month republishes it — the monthly schedule, a manual dispatch, or a push to
`main` that touches the Dockerfile. Two builds in the same month therefore leave `YYYY.MM` pointing
at the second one, and a deployment pinned to it silently changes underneath.

For a genuinely fixed image, **pin by digest** — see
[Verifying what you are running](#verifying-what-you-are-running). Use `YYYY.MM` to say *"the
August build"* when that granularity is enough.

### Included packages

| Category | Packages |
|---|---|
| Shell & base | `bash`, `cron`, `curl`, `gnupg`, `jq`, `netcat-openbsd`, `tzdata` |
| System support | `apt-utils`, `ca-certificates`, `locales` |
| Init & supervision | `s6-overlay` 3.2.3.2 (statically linked, in `/command` and `/package`) |

### Environment variables

| Variable | Default | Description |
|---|---|---|
| `PUID` | `1000` | UID applied to `appuser` at container start |
| `PGID` | `1000` | GID applied to `appuser` at container start |
| `HOME` | `/config` | Home directory of `appuser` |
| `TZ` | `UTC` | Timezone |
| `LANG` | `en_US.UTF-8` | Locale (also `LANGUAGE`, `LC_ALL`) |
| `TERM` | `xterm` | Terminal type |
| `DEBIAN_FRONTEND` | `noninteractive` | Suppresses interactive APT prompts |
| `PATH` | `/command:/usr/local/sbin:…` | `/command` holds the s6 binaries |

s6-overlay tunables set by this image:

| Variable | Value | Effect |
|---|---|---|
| `S6_BEHAVIOUR_IF_STAGE2_FAILS` | `2` | Stop the container if an init script fails |
| `S6_CMD_WAIT_FOR_SERVICES_MAXTIME` | `0` | No startup timeout imposed on services |
| `S6_VERBOSITY` | `1` | Log warnings and errors only |

Useful ones you can set yourself at runtime:

| Variable | Default | When to use it |
|---|---|---|
| `S6_SERVICES_GRACETIME` | `3000` | Milliseconds to let services exit on shutdown before escalating |
| `S6_KILL_GRACETIME` | `3000` | Milliseconds between the final `SIGTERM` and `SIGKILL` |
| `S6_READ_ONLY_ROOT` | `0` | Set to `1` when running with a read-only root filesystem |
| `S6_VERBOSITY` | `1` | Raise to `2`+ to debug the startup sequence |

The full list is in the
[s6-overlay documentation](https://github.com/just-containers/s6-overlay#customizing-s6-overlays-behaviour).

### Filesystem layout

| Path | Purpose |
|---|---|
| `/config` | Home of `appuser`, mode `750`, owned by `appuser` — mount your persistent data here |
| `/command` | s6 binaries (`s6-setuidgid`, `with-contenv`, `s6-rc`, …) |
| `/etc/s6-overlay/s6-rc.d/` | Service definitions |
| `/etc/s6-overlay/user-bundles.d/user/contents.d/` | Services enabled at boot |
| `/etc/s6-overlay/scripts/` | Shell scripts called by service definitions |

---

## Getting started

### Run a container

```bash
docker run -it --rm ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic-s6:latest
```

You get a shell as `appuser` — the default `CMD` drops privileges.

### Set the user's UID/GID

Match the container user to a host user so bind-mounted files have the right ownership:

```bash
docker run --rm \
  -e PUID=$(id -u) -e PGID=$(id -g) \
  -v "$PWD/data:/config" \
  ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic-s6:latest \
  sh -c 'id appuser'
```

`appuser` is remapped and `/config` is re-owned **before** any service starts. Values must be
integers greater than or equal to `1`; anything else stops the container with an explicit error.
`0` is refused on purpose — it is root's UID, and accepting it would silently turn `appuser` into
a second root account.

### Build your own image

Package installation happens at build time, as `root`:

```dockerfile
FROM ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic-s6:latest

RUN apt-get update && \
    apt-get install -y --no-install-recommends nginx && \
    rm -rf /var/lib/apt/lists/*
```

> **Do not install packages at container start** (e.g. an `apt-get install` in `command:`).
> Services run unprivileged, so APT will fail with `Permission denied` on `/var/lib/apt/lists`.
> Install at build time.

---

## Adding your own services

Service definitions live under `/etc/s6-overlay/s6-rc.d/`. Keep them in a `root/` directory in
your build context and copy the whole tree in — that is the convention this image itself uses.

### A one-shot init task

Runs once at startup, before services. Useful for generating configuration or fixing permissions.

`root/etc/s6-overlay/s6-rc.d/init-myapp/type`
```
oneshot
```

`root/etc/s6-overlay/s6-rc.d/init-myapp/up`
```
/etc/s6-overlay/scripts/init-myapp
```

`root/etc/s6-overlay/s6-rc.d/init-myapp/dependencies.d/init-adduser` — *empty file.*
Guarantees `PUID`/`PGID` are already applied when your task runs.

`root/etc/s6-overlay/user-bundles.d/user/contents.d/init-myapp` — *empty file.*
Enables the task at boot.

`root/etc/s6-overlay/scripts/init-myapp` — *must be executable.*
```sh
#!/command/with-contenv sh
set -eu

# Write logs to stderr: stdout belongs to the CMD.
echo "[init-myapp] preparing directories" >&2

mkdir -p /config/myapp
chown appuser:appuser /config/myapp
```

> The `up` file is **not** a shell script — it is a single
> [execline](https://skarnet.org/software/execline/) command line. Always delegate real logic to
> a separate script, as above.

### A supervised daemon

`root/etc/s6-overlay/s6-rc.d/myapp/type`
```
longrun
```

`root/etc/s6-overlay/s6-rc.d/myapp/run` — *must be executable.*
```sh
#!/command/with-contenv sh
exec 2>&1
# The supervisor runs as root; the daemon must not.
exec s6-setuidgid appuser /usr/bin/myapp --foreground
```

`root/etc/s6-overlay/s6-rc.d/myapp/dependencies.d/init-adduser` — *empty file.*
`root/etc/s6-overlay/user-bundles.d/user/contents.d/myapp` — *empty file.*

Two rules that matter:

- **The daemon must stay in the foreground.** A process that forks into the background looks like
  a crash to the supervisor and will be restarted in a loop.
- **Drop privileges with `s6-setuidgid`.** Everything under `s6-rc.d` starts as `root`.

### Wiring it into your Dockerfile

```dockerfile
COPY root/ /
RUN chmod 755 /etc/s6-overlay/scripts/* \
              /etc/s6-overlay/s6-rc.d/myapp/run
```

The `chmod` matters if your files come from a checkout that dropped the executable bit (a ZIP
download, or Git on Windows).

---

## Running several services

This is what an init system buys you: several processes in one container, started in the right
order, each restarted independently if it dies.

Ordering is expressed by creating an **empty file** named after the dependency inside a service's
`dependencies.d/` directory. To make an API wait for a database:

```
root/etc/s6-overlay/s6-rc.d/
├── database/
│   ├── type                        → longrun
│   ├── run                         → the database process
│   └── dependencies.d/
│       └── init-adduser            (empty)
└── api/
    ├── type                        → longrun
    ├── run                         → the API process
    └── dependencies.d/
        ├── init-adduser            (empty)
        └── database                (empty)   ← api starts after database
```

Enable both by creating empty files in the user bundle:

```
root/etc/s6-overlay/user-bundles.d/user/contents.d/database
root/etc/s6-overlay/user-bundles.d/user/contents.d/api
```

On shutdown the order reverses automatically: `api` stops first, then `database`.

> **Dependency ≠ readiness.** By default s6 considers a service started as soon as its process is
> running, not when it is ready to serve. If `api` truly cannot start before the database accepts
> connections, either make `database` send a
> [readiness notification](https://skarnet.org/software/s6/notifywhenup.html), or have `api`
> retry its connection — the latter is usually simpler and more robust.

### Controlling services at runtime

```bash
docker exec <container> s6-rc -a list          # active services
docker exec <container> s6-svstat /run/service/api
docker exec <container> s6-svc -r /run/service/api   # restart one service
```

---

## Logging

By default, everything a service writes goes to the container's output and is visible with
`docker logs`. That is the right default for most deployments — your log collector handles the
rest.

For a service that is too noisy for `docker logs`, s6 can give it a **dedicated, automatically
rotated log file**. Add a logger service consuming your service's output:

`root/etc/s6-overlay/s6-rc.d/myapp/producer-for`
```
myapp-log
```

`root/etc/s6-overlay/s6-rc.d/myapp-log/type`
```
longrun
```

`root/etc/s6-overlay/s6-rc.d/myapp-log/consumer-for`
```
myapp
```

`root/etc/s6-overlay/s6-rc.d/myapp-log/pipeline-name`
```
myapp-pipeline
```

`root/etc/s6-overlay/s6-rc.d/myapp-log/run` — *must be executable.*
```sh
#!/command/with-contenv sh
exec logutil-service /config/logs/myapp
```

Register the **pipeline** (not the individual services) in the user bundle:
`root/etc/s6-overlay/user-bundles.d/user/contents.d/myapp-pipeline` — *empty file.*

The log directory must exist and be writable by `nobody` before the logger starts — create it in
a oneshot that `myapp-log` depends on. Rotation defaults to 20 files of 1 MB each, configurable
with `S6_LOGGING_SCRIPT`.

> Make sure your service writes to **stdout** for this to capture anything — start its `run`
> script with `exec 2>&1` so stderr is captured too.

---

## Complete example NGINX

A working service, running unprivileged. Note that an unprivileged process cannot bind ports
below 1024, so NGINX listens on `8080`.

`Dockerfile`
```dockerfile
FROM ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic-s6:latest

RUN apt-get update && \
    apt-get install -y --no-install-recommends nginx && \
    rm -rf /var/lib/apt/lists/* && \
    # Listen on an unprivileged port (IPv4 and IPv6 lines both end in
    # "80 default_server;", so only that suffix is replaced).
    sed -i 's/80 default_server;/8080 default_server;/g' /etc/nginx/sites-enabled/default && \
    # The "user" directive only applies when the master runs as root.
    sed -i '/^user /d' /etc/nginx/nginx.conf && \
    # /run is not writable by appuser.
    sed -i 's#pid /run/nginx.pid;#pid /tmp/nginx.pid;#' /etc/nginx/nginx.conf && \
    # Send logs to the container's stdout/stderr.
    ln -sf /dev/stdout /var/log/nginx/access.log && \
    ln -sf /dev/stderr /var/log/nginx/error.log && \
    # Cache and temp directories must be writable by appuser.
    chown -R appuser:appuser /var/lib/nginx

COPY root/ /
RUN chmod 755 /etc/s6-overlay/s6-rc.d/nginx/run

EXPOSE 8080
```

`root/etc/s6-overlay/s6-rc.d/nginx/type`
```
longrun
```

`root/etc/s6-overlay/s6-rc.d/nginx/run`
```sh
#!/command/with-contenv sh
exec 2>&1
exec s6-setuidgid appuser nginx -g "daemon off;"
```

`root/etc/s6-overlay/s6-rc.d/nginx/dependencies.d/init-adduser` — *empty file.*
`root/etc/s6-overlay/user-bundles.d/user/contents.d/nginx` — *empty file.*

`docker-compose.yml`
```yaml
services:
  web:
    build: .
    container_name: nginx-web
    restart: unless-stopped
    ports:
      - "8080:8080"
    volumes:
      - ./html:/var/www/html
    environment:
      TZ: "Europe/Paris"
      PUID: 1000
      PGID: 1000
```

```bash
mkdir -p html && echo '<h1>It works</h1>' > html/index.html
docker compose up -d --build
curl http://localhost:8080
```

> A bind mount **replaces** the directory's contents in the image. Mounting an empty `./html`
> over `/var/www/html` removes NGINX's default page, and NGINX answers `403 Forbidden` because it
> has no `index.html` to serve. Put a file there first, as above.

---

## Security model

The container's **PID 1 runs as `root`**: s6-overlay needs those privileges to apply `PUID`/`PGID`
and to hand ownership over to unprivileged processes. Everything above that layer is designed to
minimise what actually runs privileged:

| Control | Implementation |
|---|---|
| Default `CMD` | Runs as `appuser`, not `root` |
| `root` account | Password locked (`passwd -l`), `/root` mode `700` |
| Login shell for `appuser` | `/usr/sbin/nologin` |
| SUID/SGID binaries | Stripped image-wide at build time |
| World-writable files | Write bit removed image-wide at build time |
| Default umask | `027` |
| `/config` | Mode `750`, owned by `appuser` |
| `PUID` / `PGID` | Validated at startup; `0` refused, so `appuser` cannot be remapped onto root |
| Init failure | Stops the container (`S6_BEHAVIOUR_IF_STAGE2_FAILS=2`) |
| Supply chain | Base image pinned by digest; s6 tarballs pinned by SHA256 and verified pre-extraction; CI actions pinned by commit SHA |

**Your responsibility:** anything you add under `s6-rc.d` starts as `root`. Wrap every
long-running process in `s6-setuidgid appuser` (or another unprivileged user) as shown above.

Recommended runtime hardening for your deployments:

```yaml
services:
  app:
    image: ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic-s6:latest
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    tmpfs:
      - /tmp
```

If you run with a **read-only root filesystem**, set `S6_READ_ONLY_ROOT=1` so s6-overlay writes
its runtime state to `/run`.

### Base image support status

Ubuntu 18.04 LTS left standard support on **31 May 2023**. Its public archive no longer receives
new security updates: those are published through Ubuntu Pro (ESM), which this image does not
subscribe to. Two consequences worth stating plainly:

- The monthly rebuild refreshes the image against what Ubuntu still serves for Bionic. It does
  **not** bring in fixes that only exist behind ESM, so a CVE fixed for 20.04 or 22.04 may stay
  open here indefinitely.
- Trivy itself flags this on every scan (*"This OS version is no longer supported by the
  distribution"*): Ubuntu no longer issues advisories for Bionic, so Trivy's data for it stops
  where those advisories stopped. The scans deliberately report vulnerabilities that have **no
  fix available** — on a frozen archive that is most of them, and filtering them out would leave
  a reassuringly empty report.

Use this image where an 18.04 userland is a hard requirement — legacy binaries, an old toolchain,
reproducing a historical environment. For anything new, start from a supported Ubuntu LTS. If you
must stay on 18.04 in production, add an Ubuntu Pro subscription inside your derived image.

### Verifying what you are running

Every image carries OCI provenance labels — the exact commit it was built from, and when:

```bash
docker inspect --format '{{json .Config.Labels}}' \
  ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic-s6:latest | jq
```

Pin by digest to guarantee byte-for-byte reproducibility:

```bash
docker pull ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic-s6@sha256:<digest>
```

Vulnerability reporting is covered in [`SECURITY.md`](./SECURITY.md).

---

## Troubleshooting

### Getting a shell inside a running container

```bash
docker exec -it <container> bash          # as appuser
docker exec -it -u 0 <container> bash     # as root, for debugging
docker logs <container>                   # init and service output
```

### Common problems

**`E: Could not open lock file /var/lib/apt/lists/lock (13: Permission denied)`**
APT is running unprivileged. Install packages at build time in your `Dockerfile`, not at container
start.

**`bind() to 0.0.0.0:80 failed (13: Permission denied)`**
Unprivileged processes cannot bind ports below 1024. Use a port ≥ 1024 inside the container and
remap it on the host (`-p 80:8080`).

**A service restarts endlessly**
The process is daemonising. Force foreground mode (`nginx -g "daemon off;"`, `--foreground`,
`-D FOREGROUND`, …).

**The container stops immediately at startup**
An init script failed — this is `S6_BEHAVIOUR_IF_STAGE2_FAILS=2` doing its job. `docker logs` will
show the failing script's error. Set `-e S6_VERBOSITY=2` for a more detailed startup trace.

**`[init-adduser] PUID invalide`**
`PUID` / `PGID` must be integers ≥ `1`. `0` is rejected because it is root's UID. Check for
quoting mistakes in your `.env`, and for a templated value built from an unset variable — those
tend to arrive as a literal `0`. An unset `PUID` is fine: it falls back to the `1000` default.

**A service never starts, no error shown**
It is probably not registered. Check that an empty file named after it exists in
`/etc/s6-overlay/user-bundles.d/user/contents.d/`, and confirm with
`docker exec <container> s6-rc -a list`.

**`exec: ... : Permission denied` on a service**
The `run` script is not executable. Add `RUN chmod 755 …` in your Dockerfile — ZIP downloads and
Git on Windows both drop the executable bit.

**Files created in a volume have the wrong owner**
Set `PUID`/`PGID` to the host user's IDs: `-e PUID=$(id -u) -e PGID=$(id -g)`.

**`docker stop` takes ~10 seconds**
A service is ignoring `SIGTERM`, so Docker waits for its timeout. Fix the service's signal
handling, or tune `S6_SERVICES_GRACETIME` / `S6_KILL_GRACETIME`.

---

## Maintenance

- **Images are rebuilt monthly** (1st of the month, 03:00 UTC) against the current Bionic
  archive, and can be triggered manually from the Actions tab. See
  [Base image support status](#base-image-support-status) for what that does and does not cover.
- **Vulnerabilities are scanned weekly** (Mondays, 04:00 UTC) and after every build that published
  an image, with Trivy, on both architectures. Findings with no fix available are included — see
  [Base image support status](#base-image-support-status). Full JSON reports are kept as build
  artifacts for 90 days, and every run writes a summary table to its workflow page. Results also
  go to the repository's **Security → Code scanning** tab, which requires code scanning to be
  enabled in *Settings → Code security*; when it is not, the scan still runs and says so in its
  summary.
- **The s6-overlay version and its checksums are pinned** in `Dockerfile-multi-arch` and updated
  manually — the procedure is in [`CONTRIBUTING.md`](./CONTRIBUTING.md#updating-s6-overlay).

Contributions are welcome: see [`CONTRIBUTING.md`](./CONTRIBUTING.md) and the
[`CODE_OF_CONDUCT.md`](./CODE_OF_CONDUCT.md).

---
---

# Documentation française

<p align="center">
  <a href="#docker-ubuntu-1804-lts-bionic--s6-overlay---sam-tech-lab">English version ↑</a> · <b>Français</b>
</p>

## Démarrage rapide

```bash
# Lancer un shell (en tant qu'appuser, non privilégié)
docker run -it --rm ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic-s6:latest

# Aligner l'utilisateur du conteneur sur celui de l'hôte
docker run --rm -e PUID=$(id -u) -e PGID=$(id -g) \
  -v "$PWD/data:/config" \
  ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic-s6:latest id appuser
```

Construire par-dessus :

```dockerfile
FROM ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic-s6:latest

RUN apt-get update && \
    apt-get install -y --no-install-recommends votre-paquet && \
    rm -rf /var/lib/apt/lists/*

COPY root/ /
```

**Sommaire :** [Présentation](#présentation) ·
[Déroulement du démarrage](#déroulement-du-démarrage) ·
[Référence de l'image](#référence-de-limage) · [Prise en main](#prise-en-main) ·
[Ajouter vos propres services](#ajouter-vos-propres-services) ·
[Plusieurs services](#plusieurs-services) · [Journalisation](#journalisation) ·
[Exemple complet NGINX](#exemple-complet-nginx) · [Modèle de sécurité](#modèle-de-sécurité) ·
[Dépannage](#dépannage) · [Maintenance](#maintenance-1)

---

## Présentation

Une **image de base Ubuntu 18.04 LTS (Bionic) minimale, durcie et multi-architecture**, construite
`FROM scratch` à partir du **rootfs OCI officiel d'Ubuntu**, avec
[s6-overlay](https://github.com/just-containers/s6-overlay) comme système d'init et superviseur de
processus.

Elle est conçue comme une **fondation pour vos propres images** : elle fournit un système d'init,
un utilisateur non-root dont l'UID/GID est configurable à l'exécution, et un socle système durci —
puis vous laisse travailler.

> **Architectures supportées :** `linux/amd64`, `linux/arm64`
> **Reconstructions mensuelles automatiques** intégrant ce qu'Ubuntu publie encore pour Bionic —
> lisez [État du support de l'image de base](#état-du-support-de-limage-de-base) avant tout
> déploiement en production.

### Points forts

- ✅ **Construite `FROM scratch`** depuis le rootfs OCI officiel Ubuntu — aucune couche de base tierce
- ✅ **s6-overlay en PID 1** — nettoyage des zombies, démarrage et arrêt ordonnés, signaux corrects
- ✅ **`PUID` / `PGID` configurables à l'exécution** — appliqués au démarrage, avant tout service
- ✅ **Supervision multi-services** avec dépendances déclarées et redémarrage automatique
- ✅ **Rotation des journaux par service** intégrée, via `logutil-service`
- ✅ **Non-root par défaut** — le `CMD` par défaut abandonne les privilèges vers `appuser`
- ✅ **Durcissement système** — compte `root` verrouillé, bits SUID/SGID supprimés, bits
  world-writable retirés, `umask 027`
- ✅ **Intégrité de la chaîne d'approvisionnement** — builder Alpine figé par digest, tarballs
  s6-overlay figés par SHA256 et vérifiés avant extraction, actions CI figées par SHA de commit
- ✅ **Init fail-fast** — un script d'init en échec arrête le conteneur au lieu de le laisser
  tourner dans un état incohérent
- ✅ **Optimisation APT & dpkg** — aucun paquet recommandé/suggéré, aucune traduction, cache propre
- ✅ **Vérifiée en continu** — hadolint, shellcheck, 9 tests d'intégration sur conteneur joués sur
  **les deux architectures**, et scans Trivy hebdomadaires

---

## Déroulement du démarrage

Comprendre la séquence de démarrage montre où votre propre code s'insère :

```
docker run
   │
   ├─ 1. ENTRYPOINT /init            s6-overlay devient PID 1
   │
   ├─ 2. oneshots s6-rc              init-adduser applique PUID/PGID
   │        (ordre des dépendances)  ← vos tâches d'init ici
   │
   ├─ 3. longruns s6-rc              démarrage des daemons supervisés
   │        (ordre des dépendances)  ← vos services ici
   │
   └─ 4. CMD                         exécuté comme processus normal
                                     le conteneur s'arrête quand il se termine
```

À l'arrêt (`docker stop`, ou fin du `CMD`), la séquence se déroule à l'envers : les services sont
arrêtés dans l'ordre des dépendances, puis les processus restants reçoivent `SIGTERM`, puis
`SIGKILL` après un délai de grâce.

---

## Référence de l'image

### Registres et tags

| Registre | Image | Architectures |
|---|---|---|
| GHCR | `ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic-s6:latest` | amd64 + arm64 |
| GHCR | `ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic-s6:YYYY.MM` | amd64 + arm64 |
| Docker Hub | `samtechlab/ubuntu-18.04-bionic-s6:latest` | amd64 + arm64 |
| Docker Hub | `samtechlab/ubuntu-18.04-bionic-s6:YYYY.MM` | amd64 + arm64 |

Les tags pointent vers un manifeste multi-architecture : Docker sélectionne automatiquement
l'image correspondant à la plateforme hôte. `latest` suit la reconstruction mensuelle.

**Aucun de ces tags n'est immuable.** Un tag `YYYY.MM` désigne le mois où un build a eu lieu, pas
un build en particulier : tout build de ce mois-là le republie — la planification mensuelle, un
déclenchement manuel, ou un push sur `main` touchant le Dockerfile. Deux builds dans le même mois
laissent donc `YYYY.MM` sur le second, et un déploiement épinglé dessus change sans prévenir.

Pour une image réellement figée, **épinglez par digest** — voir
[Vérifier ce que vous exécutez](#vérifier-ce-que-vous-exécutez). Utilisez `YYYY.MM` pour dire
« le build d'août » quand cette granularité suffit.

### Paquets inclus

| Catégorie | Paquets |
|---|---|
| Shell & base | `bash`, `cron`, `curl`, `gnupg`, `jq`, `netcat-openbsd`, `tzdata` |
| Outils système | `apt-utils`, `ca-certificates`, `locales` |
| Init & supervision | `s6-overlay` 3.2.3.2 (lié statiquement, dans `/command` et `/package`) |

### Variables d'environnement

| Variable | Défaut | Description |
|---|---|---|
| `PUID` | `1000` | UID appliqué à `appuser` au démarrage du conteneur |
| `PGID` | `1000` | GID appliqué à `appuser` au démarrage du conteneur |
| `HOME` | `/config` | Répertoire personnel de `appuser` |
| `TZ` | `UTC` | Fuseau horaire |
| `LANG` | `en_US.UTF-8` | Locale (également `LANGUAGE`, `LC_ALL`) |
| `TERM` | `xterm` | Type de terminal |
| `DEBIAN_FRONTEND` | `noninteractive` | Supprime les invites APT interactives |
| `PATH` | `/command:/usr/local/sbin:…` | `/command` contient les binaires s6 |

Réglages s6-overlay définis par cette image :

| Variable | Valeur | Effet |
|---|---|---|
| `S6_BEHAVIOUR_IF_STAGE2_FAILS` | `2` | Arrête le conteneur si un script d'init échoue |
| `S6_CMD_WAIT_FOR_SERVICES_MAXTIME` | `0` | Aucun délai de démarrage imposé aux services |
| `S6_VERBOSITY` | `1` | N'affiche que les avertissements et erreurs |

Variables utiles que vous pouvez définir à l'exécution :

| Variable | Défaut | Quand l'utiliser |
|---|---|---|
| `S6_SERVICES_GRACETIME` | `3000` | Millisecondes laissées aux services pour s'arrêter avant escalade |
| `S6_KILL_GRACETIME` | `3000` | Millisecondes entre le `SIGTERM` final et le `SIGKILL` |
| `S6_READ_ONLY_ROOT` | `0` | Mettre à `1` avec un système de fichiers racine en lecture seule |
| `S6_VERBOSITY` | `1` | Monter à `2`+ pour déboguer la séquence de démarrage |

La liste complète est dans la
[documentation s6-overlay](https://github.com/just-containers/s6-overlay#customizing-s6-overlays-behaviour).

### Arborescence

| Chemin | Rôle |
|---|---|
| `/config` | Home de `appuser`, mode `750`, appartenant à `appuser` — montez vos données persistantes ici |
| `/command` | Binaires s6 (`s6-setuidgid`, `with-contenv`, `s6-rc`, …) |
| `/etc/s6-overlay/s6-rc.d/` | Définitions de services |
| `/etc/s6-overlay/user-bundles.d/user/contents.d/` | Services activés au démarrage |
| `/etc/s6-overlay/scripts/` | Scripts shell appelés par les définitions de services |

---

## Prise en main

### Lancer un conteneur

```bash
docker run -it --rm ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic-s6:latest
```

Vous obtenez un shell en tant que `appuser` — le `CMD` par défaut abandonne les privilèges.

### Définir l'UID/GID de l'utilisateur

Faites correspondre l'utilisateur du conteneur à un utilisateur de l'hôte pour que les fichiers
des volumes montés aient la bonne propriété :

```bash
docker run --rm \
  -e PUID=$(id -u) -e PGID=$(id -g) \
  -v "$PWD/data:/config" \
  ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic-s6:latest \
  sh -c 'id appuser'
```

`appuser` est remappé et `/config` réattribué **avant** le démarrage de tout service. Les valeurs
doivent être des entiers supérieurs ou égaux à `1` ; toute autre valeur arrête le conteneur avec
une erreur explicite. `0` est refusé volontairement : c'est l'UID de root, et l'accepter ferait
silencieusement d'`appuser` un second compte root.

### Construire votre propre image

L'installation de paquets se fait au build, en tant que `root` :

```dockerfile
FROM ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic-s6:latest

RUN apt-get update && \
    apt-get install -y --no-install-recommends nginx && \
    rm -rf /var/lib/apt/lists/*
```

> **N'installez pas de paquets au démarrage du conteneur** (par ex. un `apt-get install` dans
> `command:`). Les services tournent sans privilèges, APT échouera donc avec `Permission denied`
> sur `/var/lib/apt/lists`. Installez au moment du build.

---

## Ajouter vos propres services

Les définitions de services vivent sous `/etc/s6-overlay/s6-rc.d/`. Conservez-les dans un
répertoire `root/` de votre contexte de build et copiez l'arborescence entière — c'est la
convention utilisée par cette image elle-même.

### Une tâche d'init (oneshot)

Exécutée une fois au démarrage, avant les services. Utile pour générer une configuration ou
corriger des permissions.

`root/etc/s6-overlay/s6-rc.d/init-myapp/type`
```
oneshot
```

`root/etc/s6-overlay/s6-rc.d/init-myapp/up`
```
/etc/s6-overlay/scripts/init-myapp
```

`root/etc/s6-overlay/s6-rc.d/init-myapp/dependencies.d/init-adduser` — *fichier vide.*
Garantit que `PUID`/`PGID` sont déjà appliqués quand votre tâche s'exécute.

`root/etc/s6-overlay/user-bundles.d/user/contents.d/init-myapp` — *fichier vide.*
Active la tâche au démarrage.

`root/etc/s6-overlay/scripts/init-myapp` — *doit être exécutable.*
```sh
#!/command/with-contenv sh
set -eu

# Écrire les traces sur stderr : stdout appartient au CMD.
echo "[init-myapp] préparation des répertoires" >&2

mkdir -p /config/myapp
chown appuser:appuser /config/myapp
```

> Le fichier `up` n'est **pas** un script shell : c'est une unique ligne de commande
> [execline](https://skarnet.org/software/execline/). Déléguez toujours la logique réelle à un
> script séparé, comme ci-dessus.

### Un daemon supervisé

`root/etc/s6-overlay/s6-rc.d/myapp/type`
```
longrun
```

`root/etc/s6-overlay/s6-rc.d/myapp/run` — *doit être exécutable.*
```sh
#!/command/with-contenv sh
exec 2>&1
# Le superviseur tourne en root ; le daemon ne doit pas.
exec s6-setuidgid appuser /usr/bin/myapp --foreground
```

`root/etc/s6-overlay/s6-rc.d/myapp/dependencies.d/init-adduser` — *fichier vide.*
`root/etc/s6-overlay/user-bundles.d/user/contents.d/myapp` — *fichier vide.*

Deux règles importantes :

- **Le daemon doit rester au premier plan.** Un processus qui se détache en arrière-plan est
  interprété comme un crash par le superviseur et sera relancé en boucle.
- **Abandonnez les privilèges avec `s6-setuidgid`.** Tout ce qui se trouve sous `s6-rc.d` démarre
  en `root`.

### Intégration dans votre Dockerfile

```dockerfile
COPY root/ /
RUN chmod 755 /etc/s6-overlay/scripts/* \
              /etc/s6-overlay/s6-rc.d/myapp/run
```

Le `chmod` est important si vos fichiers proviennent d'une récupération ayant perdu le bit
exécutable (téléchargement ZIP, ou Git sous Windows).

---

## Plusieurs services

C'est ce qu'apporte un système d'init : plusieurs processus dans un conteneur, démarrés dans le
bon ordre, chacun redémarré indépendamment s'il meurt.

L'ordre s'exprime en créant un **fichier vide** portant le nom de la dépendance dans le répertoire
`dependencies.d/` du service. Pour qu'une API attende une base de données :

```
root/etc/s6-overlay/s6-rc.d/
├── database/
│   ├── type                        → longrun
│   ├── run                         → le processus de base de données
│   └── dependencies.d/
│       └── init-adduser            (vide)
└── api/
    ├── type                        → longrun
    ├── run                         → le processus API
    └── dependencies.d/
        ├── init-adduser            (vide)
        └── database                (vide)   ← api démarre après database
```

Activez les deux en créant des fichiers vides dans le bundle utilisateur :

```
root/etc/s6-overlay/user-bundles.d/user/contents.d/database
root/etc/s6-overlay/user-bundles.d/user/contents.d/api
```

À l'arrêt, l'ordre s'inverse automatiquement : `api` s'arrête d'abord, puis `database`.

> **Dépendance ≠ disponibilité.** Par défaut, s6 considère un service démarré dès que son
> processus tourne, pas quand il est prêt à répondre. Si `api` ne peut vraiment pas démarrer avant
> que la base accepte les connexions, faites soit envoyer à `database` une
> [notification de disponibilité](https://skarnet.org/software/s6/notifywhenup.html), soit
> réessayer la connexion côté `api` — cette seconde option est généralement plus simple et plus
> robuste.

### Piloter les services à l'exécution

```bash
docker exec <conteneur> s6-rc -a list          # services actifs
docker exec <conteneur> s6-svstat /run/service/api
docker exec <conteneur> s6-svc -r /run/service/api   # redémarrer un service
```

---

## Journalisation

Par défaut, tout ce qu'écrit un service part vers la sortie du conteneur et est visible avec
`docker logs`. C'est le bon comportement dans la plupart des déploiements — votre collecteur de
logs fait le reste.

Pour un service trop bavard pour `docker logs`, s6 peut lui attribuer un **fichier de log dédié,
tourné automatiquement**. Ajoutez un service de log consommant la sortie du vôtre :

`root/etc/s6-overlay/s6-rc.d/myapp/producer-for`
```
myapp-log
```

`root/etc/s6-overlay/s6-rc.d/myapp-log/type`
```
longrun
```

`root/etc/s6-overlay/s6-rc.d/myapp-log/consumer-for`
```
myapp
```

`root/etc/s6-overlay/s6-rc.d/myapp-log/pipeline-name`
```
myapp-pipeline
```

`root/etc/s6-overlay/s6-rc.d/myapp-log/run` — *doit être exécutable.*
```sh
#!/command/with-contenv sh
exec logutil-service /config/logs/myapp
```

Enregistrez le **pipeline** (et non les services individuels) dans le bundle utilisateur :
`root/etc/s6-overlay/user-bundles.d/user/contents.d/myapp-pipeline` — *fichier vide.*

Le répertoire de logs doit exister et être accessible en écriture par `nobody` avant le démarrage
du logger — créez-le dans un oneshot dont `myapp-log` dépend. La rotation est par défaut de
20 fichiers de 1 Mo, configurable via `S6_LOGGING_SCRIPT`.

> Vérifiez que votre service écrit bien sur **stdout** pour que cela capture quelque chose —
> commencez son script `run` par `exec 2>&1` afin que stderr soit également capturé.

---

## Exemple complet NGINX

Un service fonctionnel, exécuté sans privilèges. Un processus non privilégié ne pouvant pas se
lier aux ports inférieurs à 1024, NGINX écoute sur `8080`.

`Dockerfile`
```dockerfile
FROM ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic-s6:latest

RUN apt-get update && \
    apt-get install -y --no-install-recommends nginx && \
    rm -rf /var/lib/apt/lists/* && \
    # Écouter sur un port non privilégié (les lignes IPv4 et IPv6 se
    # terminent toutes deux par "80 default_server;", seul ce suffixe
    # est remplacé).
    sed -i 's/80 default_server;/8080 default_server;/g' /etc/nginx/sites-enabled/default && \
    # La directive "user" ne s'applique que si le master tourne en root.
    sed -i '/^user /d' /etc/nginx/nginx.conf && \
    # /run n'est pas accessible en écriture par appuser.
    sed -i 's#pid /run/nginx.pid;#pid /tmp/nginx.pid;#' /etc/nginx/nginx.conf && \
    # Rediriger les logs vers stdout/stderr du conteneur.
    ln -sf /dev/stdout /var/log/nginx/access.log && \
    ln -sf /dev/stderr /var/log/nginx/error.log && \
    # Les répertoires de cache et temporaires doivent être accessibles
    # en écriture par appuser.
    chown -R appuser:appuser /var/lib/nginx

COPY root/ /
RUN chmod 755 /etc/s6-overlay/s6-rc.d/nginx/run

EXPOSE 8080
```

`root/etc/s6-overlay/s6-rc.d/nginx/type`
```
longrun
```

`root/etc/s6-overlay/s6-rc.d/nginx/run`
```sh
#!/command/with-contenv sh
exec 2>&1
exec s6-setuidgid appuser nginx -g "daemon off;"
```

`root/etc/s6-overlay/s6-rc.d/nginx/dependencies.d/init-adduser` — *fichier vide.*
`root/etc/s6-overlay/user-bundles.d/user/contents.d/nginx` — *fichier vide.*

`docker-compose.yml`
```yaml
services:
  web:
    build: .
    container_name: nginx-web
    restart: unless-stopped
    ports:
      - "8080:8080"
    volumes:
      - ./html:/var/www/html
    environment:
      TZ: "Europe/Paris"
      PUID: 1000
      PGID: 1000
```

```bash
mkdir -p html && echo '<h1>Ça marche</h1>' > html/index.html
docker compose up -d --build
curl http://localhost:8080
```

> Un montage de volume **remplace** le contenu du répertoire dans l'image. Monter un `./html`
> vide sur `/var/www/html` supprime la page par défaut de NGINX, et NGINX répond
> `403 Forbidden` faute d'`index.html` à servir. Créez d'abord un fichier, comme ci-dessus.

---

## Modèle de sécurité

Le **PID 1 du conteneur tourne en `root`** : s6-overlay a besoin de ces privilèges pour appliquer
`PUID`/`PGID` et transmettre la propriété à des processus non privilégiés. Tout ce qui se trouve
au-dessus de cette couche est conçu pour réduire ce qui s'exécute réellement avec des privilèges :

| Contrôle | Mise en œuvre |
|---|---|
| `CMD` par défaut | Exécuté en tant que `appuser`, pas `root` |
| Compte `root` | Mot de passe verrouillé (`passwd -l`), `/root` en mode `700` |
| Shell de connexion de `appuser` | `/usr/sbin/nologin` |
| Binaires SUID/SGID | Supprimés sur toute l'image au build |
| Fichiers world-writable | Bit d'écriture retiré sur toute l'image au build |
| Umask par défaut | `027` |
| `/config` | Mode `750`, appartenant à `appuser` |
| `PUID` / `PGID` | Validés au démarrage ; `0` refusé, `appuser` ne peut donc pas être remappé sur root |
| Échec d'init | Arrête le conteneur (`S6_BEHAVIOUR_IF_STAGE2_FAILS=2`) |
| Chaîne d'approvisionnement | Image de base figée par digest ; tarballs s6 figés par SHA256 et vérifiés avant extraction ; actions CI figées par SHA de commit |

**Votre responsabilité :** tout ce que vous ajoutez sous `s6-rc.d` démarre en `root`. Encapsulez
chaque processus long dans `s6-setuidgid appuser` (ou un autre utilisateur non privilégié), comme
montré plus haut.

Durcissement recommandé à l'exécution pour vos déploiements :

```yaml
services:
  app:
    image: ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic-s6:latest
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    tmpfs:
      - /tmp
```

Si vous utilisez un **système de fichiers racine en lecture seule**, définissez
`S6_READ_ONLY_ROOT=1` pour que s6-overlay écrive son état d'exécution dans `/run`.

### État du support de l'image de base

Ubuntu 18.04 LTS est sorti du support standard le **31 mai 2023**. Son archive publique ne reçoit
plus de nouvelles mises à jour de sécurité : celles-ci passent par Ubuntu Pro (ESM), auquel cette
image n'est pas abonnée. Deux conséquences à énoncer clairement :

- La reconstruction mensuelle rafraîchit l'image à partir de ce qu'Ubuntu sert encore pour Bionic.
  Elle n'apporte **pas** les correctifs qui n'existent que derrière l'ESM : une CVE corrigée pour
  20.04 ou 22.04 peut rester ouverte ici indéfiniment.
- Trivy le signale à chaque analyse (*« This OS version is no longer supported by the
  distribution »*) : Ubuntu ne publie plus d'avis de sécurité pour Bionic, et les données de Trivy
  s'arrêtent donc là où ces avis se sont arrêtés. Les analyses remontent volontairement les
  vulnérabilités **sans correctif disponible** — sur une archive figée, c'est la majorité d'entre
  elles, et les filtrer laisserait un rapport vide et faussement rassurant.

Utilisez cette image là où un userland 18.04 est une contrainte dure — binaires hérités, ancienne
chaîne de compilation, reproduction d'un environnement historique. Pour tout nouveau projet,
partez d'une LTS Ubuntu encore supportée. Si vous devez rester sur 18.04 en production, ajoutez un
abonnement Ubuntu Pro dans votre image dérivée.

### Vérifier ce que vous exécutez

Chaque image porte des labels de provenance OCI — le commit exact dont elle est issue, et sa date
de construction :

```bash
docker inspect --format '{{json .Config.Labels}}' \
  ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic-s6:latest | jq
```

Figez par digest pour garantir une reproductibilité au bit près :

```bash
docker pull ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic-s6@sha256:<digest>
```

Le signalement de vulnérabilités est décrit dans [`SECURITY.md`](./SECURITY.md).

---

## Dépannage

### Obtenir un shell dans un conteneur en cours d'exécution

```bash
docker exec -it <conteneur> bash          # en tant qu'appuser
docker exec -it -u 0 <conteneur> bash     # en tant que root, pour déboguer
docker logs <conteneur>                   # sortie de l'init et des services
```

### Problèmes courants

**`E: Could not open lock file /var/lib/apt/lists/lock (13: Permission denied)`**
APT s'exécute sans privilèges. Installez les paquets au build dans votre `Dockerfile`, pas au
démarrage du conteneur.

**`bind() to 0.0.0.0:80 failed (13: Permission denied)`**
Un processus non privilégié ne peut pas se lier aux ports inférieurs à 1024. Utilisez un port
≥ 1024 dans le conteneur et remappez-le côté hôte (`-p 80:8080`).

**Un service redémarre en boucle**
Le processus se détache en arrière-plan. Forcez le mode premier plan (`nginx -g "daemon off;"`,
`--foreground`, `-D FOREGROUND`, …).

**Le conteneur s'arrête immédiatement au démarrage**
Un script d'init a échoué — c'est `S6_BEHAVIOUR_IF_STAGE2_FAILS=2` qui joue son rôle.
`docker logs` affichera l'erreur du script fautif. Utilisez `-e S6_VERBOSITY=2` pour une trace de
démarrage détaillée.

**`[init-adduser] PUID invalide`**
`PUID` / `PGID` doivent être des entiers ≥ `1`. `0` est refusé car c'est l'UID de root. Vérifiez
les erreurs de quoting dans votre `.env`, ainsi que les valeurs construites depuis une variable
non définie : elles arrivent souvent sous la forme d'un `0` littéral. Un `PUID` absent, lui, ne
pose pas de problème : il retombe sur la valeur par défaut `1000`.

**Un service ne démarre jamais, sans erreur affichée**
Il n'est probablement pas enregistré. Vérifiez qu'un fichier vide à son nom existe dans
`/etc/s6-overlay/user-bundles.d/user/contents.d/`, et confirmez avec
`docker exec <conteneur> s6-rc -a list`.

**`exec: ... : Permission denied` sur un service**
Le script `run` n'est pas exécutable. Ajoutez `RUN chmod 755 …` dans votre Dockerfile — les
téléchargements ZIP et Git sous Windows perdent tous deux le bit exécutable.

**Les fichiers créés dans un volume ont le mauvais propriétaire**
Définissez `PUID`/`PGID` avec les identifiants de l'utilisateur hôte :
`-e PUID=$(id -u) -e PGID=$(id -g)`.

**`docker stop` met une dizaine de secondes**
Un service ignore `SIGTERM`, Docker attend donc son délai d'expiration. Corrigez la gestion des
signaux du service, ou ajustez `S6_SERVICES_GRACETIME` / `S6_KILL_GRACETIME`.

---

## Maintenance

- **Les images sont reconstruites chaque mois** (le 1er, à 03h00 UTC) à partir de l'archive
  Bionic courante, et peuvent être déclenchées manuellement depuis l'onglet Actions. Voir
  [État du support de l'image de base](#état-du-support-de-limage-de-base) pour ce que cela
  couvre — et ne couvre pas.
- **Les vulnérabilités sont scannées chaque semaine** (lundi, 04h00 UTC) et après chaque build
  ayant publié une image, avec Trivy, sur les deux architectures. Les vulnérabilités sans
  correctif disponible sont incluses — voir
  [État du support de l'image de base](#état-du-support-de-limage-de-base). Les rapports JSON
  complets sont conservés 90 jours en artefacts de build, et chaque run écrit un tableau de
  synthèse sur sa page de workflow. Les résultats vont aussi dans l'onglet
  **Security → Code scanning** du dépôt, ce qui suppose le code scanning activé dans
  *Settings → Code security* ; sinon l'analyse tourne quand même et le signale dans son résumé.
- **La version de s6-overlay et ses empreintes sont figées** dans `Dockerfile-multi-arch` et mises
  à jour manuellement — la procédure est dans
  [`CONTRIBUTING.md`](./CONTRIBUTING.md#mettre-à-jour-s6-overlay).

Les contributions sont bienvenues : voir [`CONTRIBUTING.md`](./CONTRIBUTING.md) et le
[`CODE_OF_CONDUCT.md`](./CODE_OF_CONDUCT.md).

---

## License / Licence

This project is distributed under the **MIT** license — see the [LICENSE](./LICENSE) file for more details.

Ce projet est distribué sous la licence **MIT** — consultez le fichier [LICENSE](./LICENSE) pour plus de détails.

---

## Copyright / Droit d'auteur

```text
Copyright (c) 2025 Sam Tech Lab
```
