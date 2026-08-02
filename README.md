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

> ⚠️ **Test bench / Banc d'essai** — This repository is an experimental variant of
> [`Docker-Ubuntu-18.04-Bionic`](https://github.com/Sam-Tech-Lab-Git/Docker-Ubuntu-18.04-Bionic),
> adding [s6-overlay](https://github.com/just-containers/s6-overlay) as PID 1. It is **not** a
> drop-in replacement: the security model differs (see [Security trade-off](#security-trade-off)).

---

## Overview

Same **minimal, hardened, multi-arch Ubuntu 18.04 (Bionic) base** as the parent image — built
`FROM scratch` from the **official Ubuntu OCI rootfs** — with
[s6-overlay](https://github.com/just-containers/s6-overlay) added as a proper init system and
process supervisor.

> Supported architectures: **`amd64`** and **`arm64`**

### What s6-overlay adds

| Capability | Without s6 (parent image) | With s6-overlay (this image) |
|---|---|---|
| **`PUID`/`PGID`** | Fixed at build time — values passed at runtime are **ignored** | **Actually applied at runtime**, before any service starts |
| **PID 1 behaviour** | The `CMD` runs as PID 1 and rarely reaps zombies or handles `SIGTERM` correctly | Real init: zombie reaping, ordered startup/shutdown, correct signal handling |
| **Multiple processes** | One `CMD` only | Any number of supervised services, with declared dependencies |
| **Init tasks** | None | `s6-rc` oneshots run before services, in dependency order |

---

## Security trade-off

This is the one thing to understand before using this image.

**s6-overlay must start as `root` (PID 1)** in order to remap `PUID`/`PGID` and then drop
privileges. This image therefore has **no `USER` directive**, unlike the parent image which ends
with `USER appuser`.

The upstream s6-overlay documentation is explicit that the privilege-changing tooling
(`fix-attrs`, `logutil-service`) does not work under a `USER` container — dynamic `PUID`/`PGID`
and a non-root PID 1 are mutually exclusive. **You cannot have both.**

What is done to mitigate it:

- The **default `CMD` drops privileges**: `docker run -it <image>` gives you an `appuser` shell,
  not a root shell.
- All other hardening from the parent image is kept: `root` account locked, SUID/SGID bits
  stripped, world-writable bits removed, `umask 027`.
- Your own services **must** drop privileges explicitly — see
  [Writing a service](#4-writing-a-supervised-service).

**If you do not need dynamic `PUID`/`PGID`**, prefer the parent image
[`Docker-Ubuntu-18.04-Bionic`](https://github.com/Sam-Tech-Lab-Git/Docker-Ubuntu-18.04-Bionic):
it runs as a non-root user end to end, which is the stronger posture.

---

## How this differs from the LinuxServer.io base image

This image takes the same broad approach as
[`linuxserver/docker-baseimage-ubuntu`](https://github.com/linuxserver/docker-baseimage-ubuntu)
but deliberately diverges on several points:

| Point | LinuxServer.io | This image |
|---|---|---|
| **Multi-arch** | `ENV ARCH=amd64` hardcoded; a separate Dockerfile per architecture | Single Dockerfile; `TARGETARCH` → s6 arch mapping resolved at build time |
| **Download integrity** | s6 tarballs fetched with no verification | **SHA256 pinned in the Dockerfile** and verified before extraction — a tampered tarball fails the build |
| **Runtime remote code** | `S6_STAGE2_HOOK=/docker-mods` downloads and executes scripts from the internet at container start | **Not used.** No remote code is fetched or executed at runtime |
| **Build-time remote deps** | Helper scripts `ADD`ed from `raw.githubusercontent.com` at mutable refs (`v3`, `v1`) | Only the pinned, checksum-verified s6 release tarballs |
| **Init failure handling** | `S6_BEHAVIOUR_IF_STAGE2_FAILS` left at default `0` — a failed init is ignored | Set to **`2`**: a failed init **stops the container** instead of running with wrong permissions |
| **System hardening** | Not applied in the base | `root` locked, SUID/SGID stripped, world-writable bits removed, `umask 027` |
| **Default `CMD`** | None (root by default) | Drops privileges to `appuser` |
| **Testing** | — | CI runs **8 integration tests** against a real container on every build |

---

## s6 environment variables

Beyond the parent image's variables, this image sets:

| Variable | Value | Why |
|---|---|---|
| `S6_BEHAVIOUR_IF_STAGE2_FAILS` | `2` | Stop the container if an init script fails, rather than continuing silently |
| `S6_CMD_WAIT_FOR_SERVICES_MAXTIME` | `0` | No maximum startup time imposed on services |
| `S6_VERBOSITY` | `1` | Only warnings and errors |

`PUID` / `PGID` (default `1000`) are now **read at container startup**, not baked in.
The full list of s6 tunables is in the
[upstream documentation](https://github.com/just-containers/s6-overlay#customizing-s6-overlays-behaviour).

---

## Usage

### 1. Run an interactive container

```bash
docker run -it --rm ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic-s6:latest
```

You land in a shell as `appuser`, not `root` — the default `CMD` drops privileges.

### 2. Verify dynamic PUID/PGID

```bash
docker run --rm -e PUID=1500 -e PGID=1600 \
  ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic-s6:latest \
  sh -c 'id appuser'
```

```
uid=1500(appuser) gid=1600(appuser) groups=1600(appuser)
```

This is the behaviour the parent image cannot provide.

### 3. Installing packages in a derived image

Package installation still happens **at build time, as `root`** — same as the parent image:

```dockerfile
FROM ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic-s6:latest

RUN apt-get update && \
    apt-get install -y --no-install-recommends nginx && \
    rm -rf /var/lib/apt/lists/*
```

No `USER root` is needed here — this image already builds as root.

### 4. Writing a supervised service

Create an `s6-rc` service definition. NGINX is used as the example:

`root/etc/s6-overlay/s6-rc.d/nginx/type`:
```
longrun
```

`root/etc/s6-overlay/s6-rc.d/nginx/run`:
```sh
#!/command/with-contenv sh
exec 2>&1
# Drop privileges: the supervisor runs as root, the service must not.
exec s6-setuidgid appuser nginx -g "daemon off;"
```

`root/etc/s6-overlay/s6-rc.d/nginx/dependencies.d/init-adduser` — empty file.
Guarantees `PUID`/`PGID` are applied **before** NGINX starts.

`root/etc/s6-overlay/user-bundles.d/user/contents.d/nginx` — empty file.
Registers the service so it starts at container boot.

Then in your Dockerfile:
```dockerfile
COPY root/ /
RUN chmod +x /etc/s6-overlay/s6-rc.d/nginx/run
```

> Note: NGINX must listen on a port ≥ 1024 (e.g. `8080`) since it no longer runs as root.

### 5. Docker Compose

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

---

## Maintaining the s6-overlay version

The s6-overlay version **and its SHA256 checksums** are pinned in `Dockerfile-multi-arch`.
Dependabot does **not** track them (it is not a package manifest it understands), so bumping the
version is a manual step. The procedure is documented in
[`CONTRIBUTING.md`](./CONTRIBUTING.md#mettre-à-jour-s6-overlay).

---

## Tags

| Registry | Tag | Architecture |
|---|---|---|
| GHCR | `ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic-s6:latest` | amd64 + arm64 |
| GHCR | `ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic-s6:YYYY.MM` | amd64 + arm64 |
| Docker Hub | `samtechlab/ubuntu-18.04-bionic-s6:latest` | amd64 + arm64 |

> Docker Hub publishing is **optional** in this repository: if the `DOCKERHUB_USERNAME` /
> `DOCKERHUB_TOKEN` secrets are absent, that step is skipped and only GHCR is published.

For the security baseline see [`SECURITY.md`](./SECURITY.md), and to contribute see
[`CONTRIBUTING.md`](./CONTRIBUTING.md) and the [`CODE_OF_CONDUCT.md`](./CODE_OF_CONDUCT.md).

---
---

## Présentation

Même **base Ubuntu 18.04 (Bionic) minimale, durcie et multi-architecture** que l'image parente —
construite `FROM scratch` à partir du **rootfs OCI officiel Ubuntu** — avec
[s6-overlay](https://github.com/just-containers/s6-overlay) ajouté comme véritable système d'init
et superviseur de processus.

> Architectures supportées : **`amd64`** et **`arm64`**

### Ce qu'apporte s6-overlay

| Fonctionnalité | Sans s6 (image parente) | Avec s6-overlay (cette image) |
|---|---|---|
| **`PUID`/`PGID`** | Figés au build — les valeurs passées à l'exécution sont **ignorées** | **Réellement appliqués à l'exécution**, avant tout service |
| **Comportement PID 1** | Le `CMD` tourne en PID 1 et gère rarement les zombies ou `SIGTERM` correctement | Véritable init : nettoyage des zombies, démarrage/arrêt ordonnés, signaux corrects |
| **Plusieurs processus** | Un seul `CMD` | Autant de services supervisés que voulu, avec dépendances déclarées |
| **Tâches d'init** | Aucune | Oneshots `s6-rc` exécutés avant les services, dans l'ordre des dépendances |

---

## Compromis de sécurité

C'est le point à comprendre avant d'utiliser cette image.

**s6-overlay doit démarrer en tant que `root` (PID 1)** pour pouvoir remapper `PUID`/`PGID` puis
abandonner ses privilèges. Cette image n'a donc **aucune directive `USER`**, contrairement à
l'image parente qui se termine par `USER appuser`.

La documentation officielle de s6-overlay est explicite : l'outillage de changement de privilèges
(`fix-attrs`, `logutil-service`) ne fonctionne pas dans un conteneur `USER` — un `PUID`/`PGID`
dynamique et un PID 1 non-root sont mutuellement exclusifs. **On ne peut pas avoir les deux.**

Ce qui est mis en place pour limiter le risque :

- Le **`CMD` par défaut abandonne les privilèges** : `docker run -it <image>` ouvre un shell
  `appuser`, pas un shell root.
- Tout le durcissement de l'image parente est conservé : compte `root` verrouillé, bits SUID/SGID
  supprimés, bits world-writable retirés, `umask 027`.
- Vos propres services **doivent** abandonner leurs privilèges explicitement — voir
  [Écrire un service](#4-écrire-un-service-supervisé).

**Si vous n'avez pas besoin de `PUID`/`PGID` dynamiques**, préférez l'image parente
[`Docker-Ubuntu-18.04-Bionic`](https://github.com/Sam-Tech-Lab-Git/Docker-Ubuntu-18.04-Bionic) :
elle tourne en utilisateur non-root de bout en bout, ce qui reste la posture la plus sûre.

---

## Différences avec l'image de base LinuxServer.io

Cette image suit la même approche générale que
[`linuxserver/docker-baseimage-ubuntu`](https://github.com/linuxserver/docker-baseimage-ubuntu)
mais s'en écarte volontairement sur plusieurs points :

| Point | LinuxServer.io | Cette image |
|---|---|---|
| **Multi-arch** | `ENV ARCH=amd64` en dur ; un Dockerfile distinct par architecture | Un seul Dockerfile ; correspondance `TARGETARCH` → arch s6 résolue au build |
| **Intégrité des téléchargements** | Tarballs s6 récupérés sans aucune vérification | **SHA256 figés dans le Dockerfile** et vérifiés avant extraction — un tarball altéré fait échouer le build |
| **Code distant à l'exécution** | `S6_STAGE2_HOOK=/docker-mods` télécharge et exécute des scripts depuis Internet au démarrage | **Non utilisé.** Aucun code distant n'est récupéré ni exécuté à l'exécution |
| **Dépendances distantes au build** | Scripts `ADD`és depuis `raw.githubusercontent.com` sur des refs mutables (`v3`, `v1`) | Uniquement les tarballs de release s6, figés et vérifiés |
| **Échec d'un script d'init** | `S6_BEHAVIOUR_IF_STAGE2_FAILS` laissé au défaut `0` — un échec est ignoré | Réglé à **`2`** : un échec **arrête le conteneur** au lieu de tourner avec de mauvaises permissions |
| **Durcissement système** | Non appliqué dans l'image de base | `root` verrouillé, SUID/SGID supprimés, world-writable retirés, `umask 027` |
| **`CMD` par défaut** | Aucun (root par défaut) | Abandonne les privilèges vers `appuser` |
| **Tests** | — | La CI exécute **8 tests d'intégration** sur un conteneur réel à chaque build |

---

## Variables d'environnement s6

En plus des variables de l'image parente, cette image définit :

| Variable | Valeur | Raison |
|---|---|---|
| `S6_BEHAVIOUR_IF_STAGE2_FAILS` | `2` | Arrête le conteneur si un script d'init échoue, au lieu de continuer silencieusement |
| `S6_CMD_WAIT_FOR_SERVICES_MAXTIME` | `0` | Aucun délai maximal de démarrage imposé aux services |
| `S6_VERBOSITY` | `1` | N'affiche que les avertissements et erreurs |

`PUID` / `PGID` (défaut `1000`) sont désormais **lus au démarrage du conteneur**, plus figés à
la construction. La liste complète des réglages s6 est dans la
[documentation officielle](https://github.com/just-containers/s6-overlay#customizing-s6-overlays-behaviour).

---

## Utilisation

### 1. Lancer un conteneur interactif

```bash
docker run -it --rm ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic-s6:latest
```

Vous arrivez dans un shell `appuser`, pas `root` — le `CMD` par défaut abandonne les privilèges.

### 2. Vérifier le PUID/PGID dynamique

```bash
docker run --rm -e PUID=1500 -e PGID=1600 \
  ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic-s6:latest \
  sh -c 'id appuser'
```

```
uid=1500(appuser) gid=1600(appuser) groups=1600(appuser)
```

C'est le comportement que l'image parente ne peut pas offrir.

### 3. Installer des paquets dans une image dérivée

L'installation de paquets se fait toujours **au build, en tant que `root`** — comme pour l'image
parente :

```dockerfile
FROM ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic-s6:latest

RUN apt-get update && \
    apt-get install -y --no-install-recommends nginx && \
    rm -rf /var/lib/apt/lists/*
```

Pas besoin de `USER root` ici : cette image construit déjà en root.

### 4. Écrire un service supervisé

Créez une définition de service `s6-rc`. Exemple avec NGINX :

`root/etc/s6-overlay/s6-rc.d/nginx/type` :
```
longrun
```

`root/etc/s6-overlay/s6-rc.d/nginx/run` :
```sh
#!/command/with-contenv sh
exec 2>&1
# Abandon des privilèges : le superviseur tourne en root, pas le service.
exec s6-setuidgid appuser nginx -g "daemon off;"
```

`root/etc/s6-overlay/s6-rc.d/nginx/dependencies.d/init-adduser` — fichier vide.
Garantit que `PUID`/`PGID` sont appliqués **avant** le démarrage de NGINX.

`root/etc/s6-overlay/user-bundles.d/user/contents.d/nginx` — fichier vide.
Enregistre le service pour qu'il démarre au boot du conteneur.

Puis dans votre Dockerfile :
```dockerfile
COPY root/ /
RUN chmod +x /etc/s6-overlay/s6-rc.d/nginx/run
```

> Remarque : NGINX doit écouter sur un port ≥ 1024 (par ex. `8080`) puisqu'il ne tourne plus
> en root.

### 5. Docker Compose

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

---

## Maintenir la version de s6-overlay

La version de s6-overlay **et ses empreintes SHA256** sont figées dans `Dockerfile-multi-arch`.
Dependabot ne les suit **pas** (ce n'est pas un manifeste de paquets qu'il sait lire) : la montée
de version est donc une opération manuelle. La procédure est documentée dans
[`CONTRIBUTING.md`](./CONTRIBUTING.md#mettre-à-jour-s6-overlay).

---

## Tags

| Registre | Tag | Architecture |
|---|---|---|
| GHCR | `ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic-s6:latest` | amd64 + arm64 |
| GHCR | `ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic-s6:YYYY.MM` | amd64 + arm64 |
| Docker Hub | `samtechlab/ubuntu-18.04-bionic-s6:latest` | amd64 + arm64 |

> La publication Docker Hub est **optionnelle** dans ce dépôt : si les secrets
> `DOCKERHUB_USERNAME` / `DOCKERHUB_TOKEN` sont absents, cette étape est ignorée et seul GHCR est
> publié.

Pour la politique de sécurité, consultez [`SECURITY.md`](./SECURITY.md) ; pour contribuer,
[`CONTRIBUTING.md`](./CONTRIBUTING.md) et le [`CODE_OF_CONDUCT.md`](./CODE_OF_CONDUCT.md).

---

## License / Licence

This project is distributed under the **MIT** license — see the [LICENSE](./LICENSE) file for more details.

Ce projet est distribué sous la licence **MIT** — consultez le fichier [LICENSE](./LICENSE) pour plus de détails.

---

## Copyright / Droit d'auteur

```text
Copyright (c) 2025 Sam Tech Lab
```
