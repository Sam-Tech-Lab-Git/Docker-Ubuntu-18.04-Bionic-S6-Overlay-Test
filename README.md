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

---

## Overview

A **minimal, hardened, multi-architecture Ubuntu 18.04 LTS (Bionic) base image**, built
`FROM scratch` from the **official Ubuntu OCI rootfs**, with
[s6-overlay](https://github.com/just-containers/s6-overlay) as its init system and process
supervisor.

It is designed as a **foundation for your own images**: it ships an init system, a non-root user
with runtime-configurable UID/GID, and a hardened system baseline — then gets out of your way.

> **Supported architectures:** `linux/amd64`, `linux/arm64`
> **Automatic monthly rebuilds** pick up the latest Ubuntu security patches.

### Key features

- ✅ **Built `FROM scratch`** from the official Ubuntu OCI rootfs — no third-party base layer
- ✅ **s6-overlay as PID 1** — zombie reaping, ordered startup and shutdown, correct signal handling
- ✅ **Runtime-configurable `PUID` / `PGID`** — applied at container start, before any service runs
- ✅ **Multi-service supervision** with declared dependencies between services
- ✅ **Non-root by default** — the default `CMD` drops privileges to `appuser`
- ✅ **System hardening** — `root` account locked, SUID/SGID bits stripped, world-writable bits
  removed, `umask 027`
- ✅ **Supply-chain integrity** — Alpine builder pinned by digest, s6-overlay tarballs pinned by
  SHA256 and verified before extraction, CI actions pinned by commit SHA
- ✅ **Fail-fast init** — a failing init script stops the container instead of running on with a
  broken state
- ✅ **APT & dpkg optimisation** — no recommended/suggested packages, no translations, clean cache
- ✅ **Continuously verified** — hadolint, shellcheck, 8 container integration tests, and weekly
  Trivy scans

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
the host platform. `YYYY.MM` tags (e.g. `2026.08`) are immutable monthly snapshots; prefer them
for reproducible deployments, and `latest` for automatic security updates.

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

Any other [s6-overlay variable](https://github.com/just-containers/s6-overlay#customizing-s6-overlays-behaviour)
can be set at runtime with `-e`.

### Filesystem layout

| Path | Purpose |
|---|---|
| `/config` | Home of `appuser`, mode `750`, owned by `appuser` — mount your persistent data here |
| `/command` | s6 binaries (`s6-setuidgid`, `with-contenv`, …) |
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
positive integers; anything else stops the container with an explicit error.

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

## Complete example: NGINX

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
| Init failure | Stops the container (`S6_BEHAVIOUR_IF_STAGE2_FAILS=2`) |
| Supply chain | Base image pinned by digest; s6 tarballs pinned by SHA256 and verified pre-extraction; CI actions pinned by commit SHA |

**Your responsibility:** anything you add under `s6-rc.d` starts as `root`. Wrap every long-running
process in `s6-setuidgid appuser` (or another unprivileged user) as shown above.

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
its runtime state to `/run` instead.

Vulnerability reporting is covered in [`SECURITY.md`](./SECURITY.md).

---

## Troubleshooting

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
show the failing script's error.

**`[init-adduser] PUID invalide`**
`PUID` / `PGID` must be positive integers. Check for quoting mistakes or empty values in your
`.env`.

**Files created in a volume have the wrong owner**
Set `PUID`/`PGID` to the host user's IDs: `-e PUID=$(id -u) -e PGID=$(id -g)`.

**Init messages appear in my piped output**
They should not — all init logging goes to stderr. Use `2>/dev/null` if you want to discard it,
and please [open an issue](https://github.com/Sam-Tech-Lab-Git/Docker-Ubuntu-18.04-Bionic-S6-Overlay-Test/issues)
if you see otherwise.

---

## Maintenance

- **Images are rebuilt monthly** (1st of the month, 03:00 UTC) with the latest Ubuntu security
  updates, and can be triggered manually from the Actions tab.
- **Vulnerabilities are scanned weekly** (Mondays, 04:00 UTC) and after every successful build,
  with Trivy. Results go to the repository's **Security → Code scanning** tab; full JSON reports
  are kept as build artifacts for 90 days.
- **The s6-overlay version and its checksums are pinned** in `Dockerfile-multi-arch` and updated
  manually — the procedure is in [`CONTRIBUTING.md`](./CONTRIBUTING.md#updating-s6-overlay).

Contributions are welcome: see [`CONTRIBUTING.md`](./CONTRIBUTING.md) and the
[`CODE_OF_CONDUCT.md`](./CODE_OF_CONDUCT.md).

---
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
> **Reconstructions mensuelles automatiques** intégrant les derniers correctifs de sécurité Ubuntu.

### Points forts

- ✅ **Construite `FROM scratch`** depuis le rootfs OCI officiel Ubuntu — aucune couche de base tierce
- ✅ **s6-overlay en PID 1** — nettoyage des zombies, démarrage et arrêt ordonnés, signaux corrects
- ✅ **`PUID` / `PGID` configurables à l'exécution** — appliqués au démarrage, avant tout service
- ✅ **Supervision multi-services** avec dépendances déclarées entre services
- ✅ **Non-root par défaut** — le `CMD` par défaut abandonne les privilèges vers `appuser`
- ✅ **Durcissement système** — compte `root` verrouillé, bits SUID/SGID supprimés, bits
  world-writable retirés, `umask 027`
- ✅ **Intégrité de la chaîne d'approvisionnement** — builder Alpine figé par digest, tarballs
  s6-overlay figés par SHA256 et vérifiés avant extraction, actions CI figées par SHA de commit
- ✅ **Init fail-fast** — un script d'init en échec arrête le conteneur au lieu de le laisser
  tourner dans un état incohérent
- ✅ **Optimisation APT & dpkg** — aucun paquet recommandé/suggéré, aucune traduction, cache propre
- ✅ **Vérifiée en continu** — hadolint, shellcheck, 8 tests d'intégration sur conteneur, et scans
  Trivy hebdomadaires

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
l'image correspondant à la plateforme hôte. Les tags `YYYY.MM` (par ex. `2026.08`) sont des
instantanés mensuels immuables — préférez-les pour des déploiements reproductibles, et `latest`
pour bénéficier automatiquement des mises à jour de sécurité.

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

Toute autre [variable s6-overlay](https://github.com/just-containers/s6-overlay#customizing-s6-overlays-behaviour)
peut être définie à l'exécution avec `-e`.

### Arborescence

| Chemin | Rôle |
|---|---|
| `/config` | Home de `appuser`, mode `750`, appartenant à `appuser` — montez vos données persistantes ici |
| `/command` | Binaires s6 (`s6-setuidgid`, `with-contenv`, …) |
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
doivent être des entiers positifs ; toute autre valeur arrête le conteneur avec une erreur
explicite.

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

## Exemple complet : NGINX

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

Le signalement de vulnérabilités est décrit dans [`SECURITY.md`](./SECURITY.md).

---

## Dépannage

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
`docker logs` affichera l'erreur du script fautif.

**`[init-adduser] PUID invalide`**
`PUID` / `PGID` doivent être des entiers positifs. Vérifiez les erreurs de quoting ou les valeurs
vides dans votre `.env`.

**Les fichiers créés dans un volume ont le mauvais propriétaire**
Définissez `PUID`/`PGID` avec les identifiants de l'utilisateur hôte :
`-e PUID=$(id -u) -e PGID=$(id -g)`.

**Les messages d'init apparaissent dans ma sortie redirigée**
Cela ne devrait pas arriver — toutes les traces d'init vont sur stderr. Utilisez `2>/dev/null`
pour les ignorer, et merci d'[ouvrir une issue](https://github.com/Sam-Tech-Lab-Git/Docker-Ubuntu-18.04-Bionic-S6-Overlay-Test/issues)
si vous constatez le contraire.

---

## Maintenance

- **Les images sont reconstruites chaque mois** (le 1er, à 03h00 UTC) avec les dernières mises à
  jour de sécurité Ubuntu, et peuvent être déclenchées manuellement depuis l'onglet Actions.
- **Les vulnérabilités sont scannées chaque semaine** (lundi, 04h00 UTC) et après chaque build
  réussi, avec Trivy. Les résultats sont dans l'onglet **Security → Code scanning** du dépôt ;
  les rapports JSON complets sont conservés 90 jours en artefacts de build.
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
