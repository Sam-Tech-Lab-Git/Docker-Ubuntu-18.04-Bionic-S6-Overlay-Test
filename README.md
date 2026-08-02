# Docker Ubuntu 18.04 LTS (Bionic) - Sam Tech Lab

<table align="center">
  <tr>
    <td align="center" width="50%">
      <a href="https://github.com/Sam-Tech-Lab-Git" target="_blank">
        <img src="https://raw.githubusercontent.com/Sam-Dz-Devops/Images/main/Sam-Tech-Site-Web.png"
             alt="Sam Tech Lab Logo" width="300"/>
      </a>
    </td>
    <td align="center" width="50%">
      <a href="https://hub.docker.com/r/samtechrepo/ubuntu-18.04-bionic" target="_blank">
        <img src="https://upload.wikimedia.org/wikipedia/commons/7/76/Ubuntu-logo-2022.svg?sanitize=true"
             alt="Ubuntu Logo" width="180"/>
      </a>
    </td>
  </tr>
</table>

<p align="center">
  <a href="https://hub.docker.com/r/samtechrepo/ubuntu-18.04-bionic" target="_blank">
    <img src="https://img.shields.io/docker/pulls/samtechrepo/ubuntu-18.04-bionic.svg?style=for-the-badge&logo=docker&logoColor=white" alt="Docker Pulls"/>
  </a>
  <a href="https://hub.docker.com/r/samtechrepo/ubuntu-18.04-bionic" target="_blank">
    <img src="https://img.shields.io/docker/stars/samtechrepo/ubuntu-18.04-bionic.svg?style=for-the-badge&logo=docker&logoColor=white" alt="Docker Stars"/>
  </a>
  <a href="https://github.com/Sam-Tech-Lab-Git" target="_blank">
    <img src="https://img.shields.io/static/v1?label=SamTechLab&message=GitHub&color=94398d&labelColor=555555&style=for-the-badge&logo=github&logoColor=white" alt="GitHub"/>
  </a>
  <a href="https://github.com/Sam-Tech-Lab-Git/Docker-Ubuntu-18.04-Bionic/blob/main/LICENSE" target="_blank">
    <img src="https://img.shields.io/badge/License-MIT-blue.svg?style=for-the-badge" alt="License: MIT"/>
  </a>
  <a href="https://github.com/Sam-Tech-Lab-Git/Docker-Ubuntu-18.04-Bionic/actions/workflows/build-multi-arch.yml" target="_blank">
      <img src="https://github.com/Sam-Tech-Lab-Git/Docker-Ubuntu-18.04-Bionic/actions/workflows/build-multi-arch.yml/badge.svg" alt="Build multi-arch - Monthly"/>
  </a>
  <a href="https://github.com/Sam-Tech-Lab-Git/Docker-Ubuntu-18.04-Bionic/actions/workflows/vuln-scan.yml" target="_blank">
      <img src="https://github.com/Sam-Tech-Lab-Git/Docker-Ubuntu-18.04-Bionic/actions/workflows/vuln-scan.yml/badge.svg" alt="Vulnerability Scan - Weekly"/>
  </a>
</p>

---

## Overview

This image provides a **clean, stable, and minimal Ubuntu 18.04 (Bionic) base** to build production-ready Docker containers.
It is built **from scratch** using the **official Ubuntu OCI rootfs**, ensuring **authenticity**, **lightness**, and **reproducibility**.

> **Automatic monthly updates**:
> The image is rebuilt every month with the latest Ubuntu updates and security patches.

Designed to be **secure, fast, and multi-purpose**, it includes advanced APT optimizations, a non-root user, and system hardening for maximum reliability.

> Supported architectures: **`amd64`** and **`arm64`**

---

## Key Features

- ✅ **"FROM scratch" image** - minimal size, built from the official Ubuntu OCI rootfs
- ✅ **Multi-arch** publishing (`amd64`, `arm64`) as a single manifest
- ✅ **APT & dpkg optimization** - no recommended packages, clean cache
- ✅ **Automatic service blocking** (`systemd`, `upstart`)
- ✅ **Non-root user (`appuser`)** for safer container execution
- ✅ **Full cleanup** of `/tmp`, `/var/log`, `/var/lib/apt/lists`
- ✅ **Locale & timezone configured** (`en_US.UTF-8`, `UTC`)
- ✅ **System hardening**:
  - `root` account locked
  - unnecessary SUID/SGID bits removed
  - default `umask 027`
- ✅ **PUID/PGID variables** for the non-root user's UID/GID
- ✅ **Automated monthly rebuilds**, linted with **hadolint**
- ✅ **Trivy** vulnerability scans (SARIF + JSON reports)

---

## Included Packages

| Category | Packages |
|---|---|
| Base utilities | `bash`, `cron`, `curl`, `gnupg`, `jq`, `netcat-openbsd`, `tzdata` |
| System support | `apt-utils`, `ca-certificates`, `locales` |

---

## Environment Variables

| Variable | Default value | Description |
|---|---|---|
| `PUID` | `1000` | Non-root user ID |
| `PGID` | `1000` | Non-root group ID |
| `HOME` | `/config` | Home directory for `appuser` |
| `LANG` | `en_US.UTF-8` | Default locale |
| `TZ` | `UTC` | Default timezone |
| `DEBIAN_FRONTEND` | `noninteractive` | Disables interactive APT prompts |

---

## Base Specifications

| Field | Value |
|---|---|
| OS | Ubuntu 18.04 (Bionic) |
| Architectures | amd64, arm64 |
| Source | Ubuntu OCI RootFS |
| Maintainer | Sam Tech Lab |
| License | MIT |
| Update frequency | Monthly (automated) |

---

## Available Tags

| Registry | Tag | Architecture |
|---|---|---|
| Docker Hub | `samtechrepo/ubuntu-18.04-bionic:latest` | amd64 + arm64 |
| Docker Hub | `samtechrepo/ubuntu-18.04-bionic:YYYY.MM` | amd64 + arm64 |
| GHCR | `ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic:latest` | amd64 + arm64 |
| GHCR | `ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic:YYYY.MM` | amd64 + arm64 |

> These tags point to a multi-architecture manifest: Docker automatically pulls the image matching the local platform.

---

## Dockerfile Source

- **Dockerfile multi-arch**: [GitHub – Sam-Tech-Lab-Git/Docker-Ubuntu-18.04-Bionic/Dockerfile-multi-arch](https://github.com/Sam-Tech-Lab-Git/Docker-Ubuntu-18.04-Bionic/blob/main/Dockerfile-multi-arch)

For the full security baseline and reporting policy, see [`SECURITY.md`](./SECURITY.md).

Want to contribute? See [`CONTRIBUTING.md`](./CONTRIBUTING.md) and the [`CODE_OF_CONDUCT.md`](./CODE_OF_CONDUCT.md).

---

## Usage Examples

### 1. Run an interactive container

```bash
docker run -it --rm samtechrepo/ubuntu-18.04-bionic:latest /bin/bash
```

### 2. Simple Dockerfile

```dockerfile
FROM samtechrepo/ubuntu-18.04-bionic:latest

# The base image runs as non-root (`USER appuser`) by default, so package
# installation must happen as root during the build, then drop back down.
USER root
RUN apt-get update && \
    apt-get install -y --no-install-recommends nginx && \
    rm -rf /var/lib/apt/lists/* && \
    sed -i 's/80 default_server;/8080 default_server;/g' /etc/nginx/sites-enabled/default && \
    sed -i '/^user /d' /etc/nginx/nginx.conf && \
    sed -i 's#pid /run/nginx.pid;#pid /tmp/nginx.pid;#' /etc/nginx/nginx.conf && \
    ln -sf /dev/stdout /var/log/nginx/access.log && \
    ln -sf /dev/stderr /var/log/nginx/error.log && \
    chown -R appuser:appuser /var/lib/nginx
USER appuser

EXPOSE 8080
CMD ["nginx", "-g", "daemon off;"]
```

This Dockerfile creates a custom image based on `ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic:latest` with NGINX preinstalled. NGINX listens on port `8080` instead of the default `80`, since binding to a privileged port (< 1024) requires root, which this container intentionally does not run as.

You can then build and test it locally:

```bash
docker build -t my-nginx .

docker run -d -p 8080:8080 my-nginx
```

> ⚠️ **Do not install packages at container startup** (e.g. via a `command:` running `apt-get install`) - since the image runs as a non-root user by default, `apt-get` will fail with `Permission denied` on `/var/lib/apt/lists`. Always install packages at **build time**, in your own Dockerfile, as shown above.

### 3. Example with Docker Compose

Use the custom Dockerfile from the previous example as the build context. Create a file named `docker-compose.yml` next to it:

```yaml
services:
  web:
    build: .
    container_name: nginx-web
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      TZ: "Europe/Paris"
```

Then start the container:

```bash
docker compose up -d
```

This builds the custom NGINX image on top of the Sam Tech Lab base, and launches the web server at http://localhost:8080.

To stop the container:

```bash
docker compose down
```

---

## Présentation

Cette image fournit une **base Ubuntu 18.04 (Bionic) propre, stable et minimaliste** pour construire des conteneurs Docker de production.
Elle est construite **from scratch** à partir du **rootfs officiel Ubuntu OCI**, garantissant **authenticité**, **légèreté** et **reproductibilité**.

> **Mises à jour automatiques mensuelles** :
> L'image est reconstruite chaque mois avec les dernières mises à jour et correctifs de sécurité Ubuntu officiels.

Conçue pour être sécurisée, rapide et multi-usage, elle inclut des optimisations APT avancées, un utilisateur non-root et un durcissement du système pour une compatibilité maximale.

> Architectures supportées : **`amd64`** et **`arm64`**

---

## Points forts

- ✅ **Image "FROM scratch"** : taille minimale, construite depuis le rootfs officiel Ubuntu OCI
- ✅ **Publication multi-arch** (`amd64`, `arm64`) sous un manifeste unique
- ✅ **Optimisation APT & dpkg** : aucun paquet recommandé, cache propre
- ✅ **Blocage des services automatiques** (`systemd`, `upstart`)
- ✅ **Utilisateur non-root (`appuser`)** pour une exécution plus sûre
- ✅ **Nettoyage complet** : `/tmp`, `/var/log`, `/var/lib/apt/lists`
- ✅ **Locale & fuseau horaire configurés** (`en_US.UTF-8`, `UTC`)
- ✅ **Durcissement système** :
  - compte `root` verrouillé
  - suppression des bits SUID/SGID inutiles
  - `umask 027` par défaut
- ✅ **Variables PUID/PGID** pour l'UID/GID de l'utilisateur non-root
- ✅ **Reconstructions mensuelles automatiques**, avec lint **hadolint**
- ✅ **Scans de vulnérabilités Trivy** (rapports SARIF + JSON)

---

## Paquets inclus

| Catégorie | Paquets |
|---|---|
| Base système | `bash`, `cron`, `curl`, `gnupg`, `jq`, `netcat-openbsd`, `tzdata` |
| Outils système | `apt-utils`, `ca-certificates`, `locales` |

---

## Variables d'environnement

| Variable | Valeur par défaut | Description |
|---|---|---|
| `PUID` | `1000` | Identifiant de l'utilisateur non-root |
| `PGID` | `1000` | Identifiant de groupe non-root |
| `HOME` | `/config` | Répertoire personnel de `appuser` |
| `LANG` | `en_US.UTF-8` | Locale par défaut |
| `TZ` | `UTC` | Fuseau horaire |
| `DEBIAN_FRONTEND` | `noninteractive` | Empêche les invites APT interactives |

---

## Spécifications de base

| Champ | Valeur |
|---|---|
| Système | Ubuntu 18.04 (Bionic) |
| Architectures | amd64, arm64 |
| Source | Ubuntu OCI RootFS |
| Mainteneur | Sam Tech Lab |
| Licence | MIT |
| Mise à jour | Mensuelle (automatisée) |

---

## Tags disponibles

| Registre | Tag | Architecture |
|---|---|---|
| Docker Hub | `samtechrepo/ubuntu-18.04-bionic:latest` | amd64 + arm64 |
| Docker Hub | `samtechrepo/ubuntu-18.04-bionic:YYYY.MM` | amd64 + arm64 |
| GHCR | `ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic:latest` | amd64 + arm64 |
| GHCR | `ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic:YYYY.MM` | amd64 + arm64 |

> Ces tags pointent vers un manifeste multi-architecture : Docker sélectionne automatiquement l'image correspondant à la plateforme locale.

---

## Source du Dockerfile

- **Dockerfile multi-arch** : [GitHub – Sam-Tech-Lab-Git/Docker-Ubuntu-18.04-Bionic/Dockerfile-multi-arch](https://github.com/Sam-Tech-Lab-Git/Docker-Ubuntu-18.04-Bionic/blob/main/Dockerfile-multi-arch)

Pour les détails de sécurité et la politique de signalement, consultez [`SECURITY.md`](./SECURITY.md).

Envie de contribuer ? Consultez [`CONTRIBUTING.md`](./CONTRIBUTING.md) et le [`CODE_OF_CONDUCT.md`](./CODE_OF_CONDUCT.md).

---

## Exemple d'utilisation

### 1. Lancer un conteneur interactif

```bash
docker run -it --rm samtechrepo/ubuntu-18.04-bionic:latest /bin/bash
```

### 2. Dockerfile simple

```dockerfile
FROM samtechrepo/ubuntu-18.04-bionic:latest

# L'image de base tourne en non-root (`USER appuser`) par défaut : l'installation
# de paquets doit donc se faire en root pendant le build, puis repasser en appuser.
USER root
RUN apt-get update && \
    apt-get install -y --no-install-recommends nginx && \
    rm -rf /var/lib/apt/lists/* && \
    sed -i 's/80 default_server;/8080 default_server;/g' /etc/nginx/sites-enabled/default && \
    sed -i '/^user /d' /etc/nginx/nginx.conf && \
    sed -i 's#pid /run/nginx.pid;#pid /tmp/nginx.pid;#' /etc/nginx/nginx.conf && \
    ln -sf /dev/stdout /var/log/nginx/access.log && \
    ln -sf /dev/stderr /var/log/nginx/error.log && \
    chown -R appuser:appuser /var/lib/nginx
USER appuser

EXPOSE 8080
CMD ["nginx", "-g", "daemon off;"]
```

Ce Dockerfile crée une image personnalisée basée sur `ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic:latest`, avec NGINX préinstallé. NGINX écoute sur le port `8080` plutôt que le port `80` par défaut, car se lier à un port privilégié (< 1024) nécessite les droits root, que ce conteneur n'a volontairement pas.

Vous pouvez ensuite la construire et la tester localement :

```bash
docker build -t my-nginx .

docker run -d -p 8080:8080 my-nginx
```

> ⚠️ **N'installez pas de paquets au démarrage du conteneur** (par exemple via un `command:` lançant `apt-get install`) - l'image tournant en utilisateur non-root par défaut, `apt-get` échouera avec `Permission denied` sur `/var/lib/apt/lists`. Installez toujours les paquets **au moment du build**, dans votre propre Dockerfile, comme montré ci-dessus.

### 3. Exemple avec Docker Compose

Utilisez le Dockerfile personnalisé de l'exemple précédent comme contexte de build. Créez un fichier nommé `docker-compose.yml` à côté :

```yaml
services:
  web:
    build: .
    container_name: nginx-web
    restart: unless-stopped
    ports:
      - "8080:8080"
    environment:
      TZ: "Europe/Paris"
```

Puis lancer le conteneur :

```bash
docker compose up -d
```

Cela construit l'image NGINX personnalisée à partir de la base Sam Tech Lab, et démarre le serveur web sur http://localhost:8080.

Arrêter le conteneur :

```bash
docker compose down
```

---

## License / Licence

This project is distributed under the **MIT** license - see the [LICENSE](https://github.com/Sam-Tech-Lab-Git/Docker-Ubuntu-18.04-Bionic/blob/main/LICENSE) file for more details.

Ce projet est distribué sous la licence **MIT** - consultez le fichier [LICENSE](https://github.com/Sam-Tech-Lab-Git/Docker-Ubuntu-18.04-Bionic/blob/main/LICENSE) pour plus de détails.

---

## Copyright / Droit d'auteur

```text
Copyright (c) 2026 Sam Tech Lab
```
