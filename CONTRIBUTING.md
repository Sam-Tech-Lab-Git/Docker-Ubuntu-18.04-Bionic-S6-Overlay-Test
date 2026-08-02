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
4. Build locally to confirm the image still builds and runs:
   ```bash
   docker buildx build -f Dockerfile-multi-arch -t ubuntu-18.04-bionic-test .
   docker run -it --rm ubuntu-18.04-bionic-test /bin/bash
   ```
5. Open a pull request against `main` describing what changed and why.

## CI on pull requests

- `build-multi-arch.yml` only builds and pushes on a push to `main` (path-filtered to `Dockerfile-multi-arch`) or via manual dispatch — **it does not run on pull requests**, so a PR with no CI activity is expected. Test locally first (see step 3–4 above).
- `vuln-scan.yml` scans the published image weekly and after successful builds — it isn't part of PR review either.

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
4. Construisez l'image localement pour confirmer qu'elle build et fonctionne toujours :
   ```bash
   docker buildx build -f Dockerfile-multi-arch -t ubuntu-18.04-bionic-test .
   docker run -it --rm ubuntu-18.04-bionic-test /bin/bash
   ```
5. Ouvrez une pull request vers `main` en décrivant ce qui a changé et pourquoi.

### CI sur les pull requests

- `build-multi-arch.yml` ne build/push que sur un push vers `main` (filtré sur `Dockerfile-multi-arch`) ou via déclenchement manuel — **il ne tourne pas sur les pull requests**, donc l'absence de CI sur une PR est normale. Testez localement d'abord (voir étapes 3–4 ci-dessus).
- `vuln-scan.yml` scanne l'image publiée chaque semaine et après un build réussi — il ne fait pas non plus partie de la revue de PR.

### Style

- Gardez la documentation bilingue (section anglaise, puis section française parallèle), comme dans `README.md` et `SECURITY.md`.
- Préférez des changements minimes et ciblés aux refontes — c'est une image de base petite et à but unique.

### Code de conduite

Ce projet suit le [Code de conduite](./CODE_OF_CONDUCT.md).
