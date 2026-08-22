# Contributing — Docker Ubuntu 18.04 (Bionic)

Thanks for your interest in improving this project.

---

## Before you start

- **Security vulnerability?** Do not open a public issue — follow the process in [`SECURITY.md`](./SECURITY.md) instead.
- **Single source of truth:** the image is built from one file, [`Dockerfile-multi-arch`](./Dockerfile-multi-arch). There are no separate per-architecture Dockerfiles — please don't reintroduce that pattern without discussing it in an issue first.

## Reporting bugs or suggesting changes

Open an issue describing:
- the affected file (`Dockerfile-multi-arch`, a workflow in `.github/workflows/`, or documentation)
- the problem or suggestion
- if relevant, the `docker build` / `docker run` output that reproduces it

## Submitting a change

1. Fork the repository and create a branch from `main`.
2. Make your change.
3. Lint the Dockerfile locally before opening a PR:
   ```bash
   docker run --rm -i ghcr.io/hadolint/hadolint:latest hadolint --failure-threshold error - < Dockerfile-multi-arch
   ```
4. Lint the s6 init scripts:
   ```bash
   shellcheck -s sh root/etc/s6-overlay/scripts/*
   ```
5. Build locally to confirm the image still builds and runs:
   ```bash
   docker buildx build -f Dockerfile-multi-arch -t ubuntu-18.04-bionic-s6-test --load .
   docker run -it --rm ubuntu-18.04-bionic-s6-test
   ```
6. Check that s6 still behaves as expected:
   ```bash
   # s6 must be PID 1
   docker run --rm ubuntu-18.04-bionic-s6-test sh -c 'cat /proc/1/comm'   # s6-svscan
   # PUID/PGID must be applied at runtime
   docker run --rm -e PUID=1500 -e PGID=1600 ubuntu-18.04-bionic-s6-test sh -c 'id appuser'
   ```
7. Open a pull request against `main` describing what changed and why.

## Updating s6-overlay

The s6-overlay version and its SHA256 checksums are pinned in `Dockerfile-multi-arch`.
Dependabot does not track them — the version lives in an `ARG`, not a dependency manifest — so
`s6-overlay-watch.yml` checks upstream every Monday and opens an issue when a newer release
exists, carrying the three official checksums with it. Updating is then this manual procedure:

1. Pick the target version from the [s6-overlay releases](https://github.com/just-containers/s6-overlay/releases).
2. Fetch the official checksums for that version:
   ```bash
   V=3.2.3.2   # replace with the target version
   for f in noarch x86_64 aarch64; do
     curl -sL "https://github.com/just-containers/s6-overlay/releases/download/v${V}/s6-overlay-${f}.tar.xz.sha256"
   done
   ```
3. Update `S6_OVERLAY_VERSION`, `S6_SHA256_NOARCH`, `S6_SHA256_X86_64` and `S6_SHA256_AARCH64`
   in `Dockerfile-multi-arch` — **all four together**. A version bumped without its checksums
   will fail the build, which is the intended safety net.
4. Update the s6-overlay badge version in `README.md`.
5. Build and run the checks above. The CI integration tests will also run on merge to `main`.

## CI on pull requests

- `build-multi-arch.yml` runs its **lint and integration-test jobs on every pull request**, so a PR that changes the Dockerfile, an s6 service or a workflow is fully checked before merge. The integration tests run as a matrix over **amd64 and arm64** — arm64 builds and runs under QEMU emulation, which costs two to three extra minutes. Publishing waits on both. It only **builds and pushes** on a push to `main` (path-filtered to `Dockerfile-multi-arch` and `root/**`), on the monthly schedule, or via manual dispatch — never from a pull request.
- Both registries are served by a **single build**, so they publish the same digest. Adding a tag means adding it to the `Composer la liste des tags` step, not adding a second build.
- Images publish an **SBOM and a `mode=max` provenance attestation** alongside the manifest.
  Inspect them with `docker buildx imagetools inspect <image> --format '{{json .SBOM}}'`.
- Do **not** quote values in the `labels:` block. Each line is passed verbatim to `--label` and buildx splits on the first `=` without stripping anything, so quotes end up inside the published value. A post-publish step fails the build if any label carries them.
- `vuln-scan.yml` scans the published image weekly, and after any build that actually published one — it isn't part of PR review either. It runs one matrixed job per architecture.

## The Docker Hub description

Docker Hub's overview is capped at **25 000 bytes**, and `README.md` is roughly twice that. The
description published there is therefore a separate, condensed file, `README-dockerhub.md`:

- It has its own workflow, `dockerhub-description.yml`, triggered when that file changes — a
  documentation edit does not need an image rebuild to reach Docker Hub. `README.md` stays the
  reference documentation on GitHub.
- Docker Hub strips most raw HTML and does not resolve repository-relative links, so this file
  uses plain Markdown and absolute GitHub URLs — keep it that way.
- Both files must stay bilingual and must agree on facts. When you change tags, environment
  variables, defaults or the security model, update both.
- The lint job fails the build if the file exceeds the limit. It measures **bytes**, not
  characters — accented text costs two bytes each in UTF-8.

## Style

- Keep documentation bilingual (English section, then a parallel French section), matching the existing `README.md` and `SECURITY.md`.
- Prefer minimal, targeted changes over refactors — this is a small, single-purpose base image.

## Code of Conduct

This project follows the [Code of Conduct](./CODE_OF_CONDUCT.md).

---

## Français

Merci de votre intérêt pour ce projet.

### Avant de commencer

- **Vulnérabilité de sécurité ?** N'ouvrez pas d'issue publique — suivez la procédure décrite dans [`SECURITY.md`](./SECURITY.md).
- **Source unique de vérité :** l'image est construite à partir d'un seul fichier, [`Dockerfile-multi-arch`](./Dockerfile-multi-arch). Il n'y a plus de Dockerfiles séparés par architecture — merci de ne pas réintroduire ce schéma sans en discuter d'abord dans une issue.

### Signaler un bug ou proposer un changement

Ouvrez une issue en précisant :
- le fichier concerné (`Dockerfile-multi-arch`, un workflow de `.github/workflows/`, ou la documentation)
- le problème ou la suggestion
- si pertinent, la sortie `docker build` / `docker run` permettant de reproduire le problème

### Proposer une modification

1. Forkez le dépôt et créez une branche depuis `main`.
2. Effectuez votre modification.
3. Vérifiez le Dockerfile localement avant d'ouvrir une PR :
   ```bash
   docker run --rm -i ghcr.io/hadolint/hadolint:latest hadolint --failure-threshold error - < Dockerfile-multi-arch
   ```
4. Vérifiez les scripts d'init s6 :
   ```bash
   shellcheck -s sh root/etc/s6-overlay/scripts/*
   ```
5. Construisez l'image localement pour confirmer qu'elle build et fonctionne toujours :
   ```bash
   docker buildx build -f Dockerfile-multi-arch -t ubuntu-18.04-bionic-s6-test --load .
   docker run -it --rm ubuntu-18.04-bionic-s6-test
   ```
6. Vérifiez que s6 se comporte toujours comme attendu :
   ```bash
   # s6 doit être PID 1
   docker run --rm ubuntu-18.04-bionic-s6-test sh -c 'cat /proc/1/comm'   # s6-svscan
   # PUID/PGID doivent être appliqués à l'exécution
   docker run --rm -e PUID=1500 -e PGID=1600 ubuntu-18.04-bionic-s6-test sh -c 'id appuser'
   ```
7. Ouvrez une pull request vers `main` en décrivant ce qui a changé et pourquoi.

### Mettre à jour s6-overlay

La version de s6-overlay et ses empreintes SHA256 sont figées dans `Dockerfile-multi-arch`.
Dependabot ne les suit pas — la version est dans un `ARG`, pas dans un manifeste de dépendances —
aussi `s6-overlay-watch.yml` interroge l'amont chaque lundi et ouvre une issue dès qu'une release
plus récente existe, en y joignant les trois empreintes officielles. La mise à jour reste ensuite
manuelle :

1. Choisissez la version cible dans les [releases s6-overlay](https://github.com/just-containers/s6-overlay/releases).
2. Récupérez les empreintes officielles de cette version :
   ```bash
   V=3.2.3.2   # remplacer par la version cible
   for f in noarch x86_64 aarch64; do
     curl -sL "https://github.com/just-containers/s6-overlay/releases/download/v${V}/s6-overlay-${f}.tar.xz.sha256"
   done
   ```
3. Mettez à jour `S6_OVERLAY_VERSION`, `S6_SHA256_NOARCH`, `S6_SHA256_X86_64` et
   `S6_SHA256_AARCH64` dans `Dockerfile-multi-arch` — **les quatre ensemble**. Une version
   montée sans ses empreintes fera échouer le build, ce qui est le filet de sécurité voulu.
4. Mettez à jour la version du badge s6-overlay dans `README.md`.
5. Rejouez les vérifications ci-dessus. Les tests d'intégration de la CI tourneront aussi au
   merge sur `main`.

### CI sur les pull requests

- `build-multi-arch.yml` exécute ses **jobs de lint et de tests d'intégration sur chaque pull request** : une PR modifiant le Dockerfile, un service s6 ou un workflow est donc entièrement vérifiée avant merge. Les tests d'intégration tournent en matrice sur **amd64 et arm64** — l'arm64 est construit et exécuté sous émulation QEMU, pour deux à trois minutes de plus. La publication attend les deux. Il ne **build et ne publie** que sur un push vers `main` (filtré sur `Dockerfile-multi-arch` et `root/**`), sur la planification mensuelle, ou via déclenchement manuel — jamais depuis une pull request.
- Les deux registres sont servis par un **build unique**, afin qu'ils publient le même digest. Ajouter un tag consiste à l'ajouter à l'étape `Composer la liste des tags`, pas à ajouter un second build.
- Les images publient un **SBOM et une attestation de provenance `mode=max`** à côté du manifeste.
  Pour les consulter : `docker buildx imagetools inspect <image> --format '{{json .SBOM}}'`.
- N'encadrez **pas** les valeurs du bloc `labels:` de guillemets. Chaque ligne est passée telle quelle à `--label`, et buildx découpe au premier `=` sans rien retirer : les guillemets se retrouvent dans la valeur publiée. Une étape post-publication fait échouer le build si un label en porte.
- `vuln-scan.yml` scanne l'image publiée chaque semaine, et après tout build en ayant réellement publié une — il ne fait pas non plus partie de la revue de PR. Il tourne en matrice, un job par architecture.

### La description Docker Hub

La description affichée sur Docker Hub est plafonnée à **25 000 octets**, et `README.md` en fait
environ le double. Ce qui est publié là-bas est donc un fichier distinct et condensé,
`README-dockerhub.md` :

- Il a son propre workflow, `dockerhub-description.yml`, déclenché quand ce fichier change : une
  modification de documentation n'a pas besoin d'une reconstruction d'image pour atteindre Docker
  Hub. `README.md` reste la documentation de référence sur GitHub.
- Docker Hub retire la plupart du HTML brut et ne résout pas les liens relatifs au dépôt : ce
  fichier n'utilise donc que du Markdown simple et des URL GitHub absolues — conservez-le ainsi.
- Les deux fichiers doivent rester bilingues et rester d'accord sur les faits. Quand vous modifiez
  les tags, les variables d'environnement, les valeurs par défaut ou le modèle de sécurité,
  mettez les deux à jour.
- Le job de lint fait échouer le build si le fichier dépasse la limite. Il mesure des **octets**,
  pas des caractères — un accent en coûte deux en UTF-8.

### Style

- Gardez la documentation bilingue (section anglaise, puis section française parallèle), comme dans `README.md` et `SECURITY.md`.
- Préférez des changements minimes et ciblés aux refontes — c'est une image de base petite et à but unique.

### Code de conduite

Ce projet suit le [Code de conduite](./CODE_OF_CONDUCT.md).
