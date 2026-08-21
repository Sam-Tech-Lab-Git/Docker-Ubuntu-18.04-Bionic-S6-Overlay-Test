# Ubuntu 18.04 LTS (Bionic) + s6-overlay

Minimal, hardened, multi-architecture **Ubuntu 18.04 LTS (Bionic)** base image, built `FROM
scratch` from the official Ubuntu OCI rootfs, with
[s6-overlay](https://github.com/just-containers/s6-overlay) as its init system and process
supervisor.

It is a **foundation for your own images**: it ships an init system, a non-root user whose UID/GID
is configurable at runtime, and a hardened system baseline — then gets out of your way.

**Full documentation, in English and French:**
<https://github.com/Sam-Tech-Lab-Git/Docker-Ubuntu-18.04-Bionic-S6-Overlay-Test>

*Version française plus bas.*

---

## ⚠️ Read this first — Ubuntu 18.04 is out of standard support

Ubuntu 18.04 LTS left standard support on **31 May 2023**. Its public archive no longer receives
new security updates: those are published through Ubuntu Pro (ESM), which this image does not
subscribe to.

- Monthly rebuilds refresh the image against what Ubuntu still serves for Bionic. They do **not**
  bring in fixes that exist only behind ESM, so a CVE fixed for 20.04 or 22.04 may stay open here
  indefinitely.
- The published vulnerability scans deliberately include findings with **no fix available** —
  on a frozen archive that is most of them. The reports are long on purpose.

Use this image where an 18.04 userland is a hard requirement: legacy binaries, an old toolchain,
reproducing a historical environment. **For anything new, start from a supported Ubuntu LTS.** If
you must run 18.04 in production, add an Ubuntu Pro subscription inside your derived image.

---

## Tags

| Tag | Contents |
|---|---|
| `latest` | Tracks the monthly rebuild — amd64 + arm64 |
| `YYYY.MM` (e.g. `2026.08`) | Immutable monthly snapshot — amd64 + arm64 |

Tags point at a multi-architecture manifest; Docker selects the right image for the host platform.
Prefer `YYYY.MM` for reproducible deployments.

Also published on GHCR as `ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic-s6`.

---

## Quick start

```bash
# Shell as the unprivileged appuser
docker run -it --rm samtechlab/ubuntu-18.04-bionic-s6:latest

# Match the container user to your host user
docker run --rm -e PUID=$(id -u) -e PGID=$(id -g) \
  -v "$PWD/data:/config" \
  samtechlab/ubuntu-18.04-bionic-s6:latest id appuser
```

Build on top of it — install packages at **build time**, as root:

```dockerfile
FROM samtechlab/ubuntu-18.04-bionic-s6:latest

RUN apt-get update && \
    apt-get install -y --no-install-recommends your-package && \
    rm -rf /var/lib/apt/lists/*

COPY root/ /
```

> Do **not** install packages at container start. Services run unprivileged, so APT fails with
> `Permission denied` on `/var/lib/apt/lists`.

---

## How the container boots

```
docker run
   │
   ├─ 1. ENTRYPOINT /init      s6-overlay takes PID 1
   ├─ 2. s6-rc oneshots        init-adduser applies PUID/PGID   ← your init tasks
   ├─ 3. s6-rc longruns        supervised daemons start         ← your services
   └─ 4. CMD                   container exits when it exits
```

On shutdown the sequence reverses: services stop in dependency order, then remaining processes
get `SIGTERM`, then `SIGKILL` after a grace period.

---

## Key features

- Built `FROM scratch` from the official Ubuntu OCI rootfs — no third-party base layer
- **s6-overlay as PID 1** — zombie reaping, ordered startup and shutdown, correct signal handling
- **Runtime-configurable `PUID` / `PGID`** — applied before any service runs
- **Multi-service supervision** with declared dependencies and automatic restart
- Per-service log rotation via `logutil-service`
- **Non-root by default** — the default `CMD` drops privileges to `appuser`
- **Hardening** — `root` locked, SUID/SGID stripped, world-writable bits removed, `umask 027`
- **Supply-chain integrity** — Alpine builder pinned by digest, s6-overlay tarballs pinned by
  SHA256 and verified before extraction, CI actions pinned by commit SHA
- **Fail-fast init** — a failing init script stops the container rather than running on broken
- Continuously verified — hadolint, shellcheck, 9 container integration tests, weekly Trivy scans

---

## Environment variables

| Variable | Default | Description |
|---|---|---|
| `PUID` | `1000` | UID applied to `appuser` at container start |
| `PGID` | `1000` | GID applied to `appuser` at container start |
| `HOME` | `/config` | Home directory of `appuser` |
| `TZ` | `UTC` | Timezone |
| `LANG` | `en_US.UTF-8` | Locale (also `LANGUAGE`, `LC_ALL`) |
| `TERM` | `xterm` | Terminal type |

`PUID` / `PGID` must be integers **≥ 1**. `0` is refused on purpose — it is root's UID, and
accepting it would silently turn `appuser` into a second root account. Anything invalid stops the
container with an explicit error.

s6-overlay tunables set by this image, and the useful ones you can set yourself:

| Variable | Value | Effect |
|---|---|---|
| `S6_BEHAVIOUR_IF_STAGE2_FAILS` | `2` | Stop the container if an init script fails |
| `S6_CMD_WAIT_FOR_SERVICES_MAXTIME` | `0` | No startup timeout imposed on services |
| `S6_VERBOSITY` | `1` | Warnings and errors only — raise to `2`+ to debug startup |
| `S6_SERVICES_GRACETIME` | `3000` | Milliseconds for services to exit on shutdown |
| `S6_KILL_GRACETIME` | `3000` | Milliseconds between the final `SIGTERM` and `SIGKILL` |
| `S6_READ_ONLY_ROOT` | `0` | Set to `1` with a read-only root filesystem |

---

## Filesystem layout

| Path | Purpose |
|---|---|
| `/config` | Home of `appuser`, mode `750` — mount your persistent data here |
| `/command` | s6 binaries (`s6-setuidgid`, `with-contenv`, `s6-rc`, …) |
| `/etc/s6-overlay/s6-rc.d/` | Service definitions |
| `/etc/s6-overlay/user-bundles.d/user/contents.d/` | Services enabled at boot |
| `/etc/s6-overlay/scripts/` | Shell scripts called by service definitions |

---

## Adding a supervised service

Keep your service tree in a `root/` directory in the build context and copy it in whole.

`root/etc/s6-overlay/s6-rc.d/myapp/type` → `longrun`

`root/etc/s6-overlay/s6-rc.d/myapp/run` — *must be executable*
```sh
#!/command/with-contenv sh
exec 2>&1
# The supervisor runs as root; the daemon must not.
exec s6-setuidgid appuser /usr/bin/myapp --foreground
```

`root/etc/s6-overlay/s6-rc.d/myapp/dependencies.d/init-adduser` — *empty file*
`root/etc/s6-overlay/user-bundles.d/user/contents.d/myapp` — *empty file*

```dockerfile
COPY root/ /
RUN chmod 755 /etc/s6-overlay/s6-rc.d/myapp/run
```

Two rules that matter:

- **The daemon must stay in the foreground.** A process that forks into the background looks like
  a crash to the supervisor and is restarted in a loop.
- **Drop privileges with `s6-setuidgid`.** Everything under `s6-rc.d` starts as `root`.

Ordering between services, per-service logging, and a complete working NGINX example are covered
in the [full documentation](https://github.com/Sam-Tech-Lab-Git/Docker-Ubuntu-18.04-Bionic-S6-Overlay-Test#adding-your-own-services).

---

## Security model

PID 1 runs as `root`: s6-overlay needs those privileges to apply `PUID`/`PGID` and hand ownership
to unprivileged processes. Runtime-configurable UID/GID and a non-root PID 1 are mutually
exclusive. Everything above that layer is designed to minimise what actually runs privileged.

| Control | Implementation |
|---|---|
| Default `CMD` | Runs as `appuser`, not `root` |
| `root` account | Password locked, `/root` mode `700` |
| Login shell for `appuser` | `/usr/sbin/nologin` |
| SUID/SGID binaries | Stripped image-wide at build time |
| World-writable files | Write bit removed image-wide at build time |
| Default umask | `027` |
| `/config` | Mode `750`, owned by `appuser` |
| `PUID` / `PGID` | Validated at startup; `0` refused |
| Init failure | Stops the container |

**Your responsibility:** anything you add under `s6-rc.d` starts as `root`. Wrap every
long-running process in `s6-setuidgid appuser`.

Recommended runtime hardening:

```yaml
services:
  app:
    image: samtechlab/ubuntu-18.04-bionic-s6:latest
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    tmpfs:
      - /tmp
```

Vulnerability reporting:
[SECURITY.md](https://github.com/Sam-Tech-Lab-Git/Docker-Ubuntu-18.04-Bionic-S6-Overlay-Test/blob/main/SECURITY.md)

---

## Troubleshooting

**`E: Could not open lock file /var/lib/apt/lists/lock (13: Permission denied)`**
APT is running unprivileged. Install packages at build time in your `Dockerfile`.

**`bind() to 0.0.0.0:80 failed (13: Permission denied)`**
Unprivileged processes cannot bind ports below 1024. Use a port ≥ 1024 inside the container and
remap it on the host (`-p 80:8080`).

**A service restarts endlessly**
The process is daemonising. Force foreground mode (`nginx -g "daemon off;"`, `--foreground`, …).

**The container stops immediately at startup**
An init script failed — `S6_BEHAVIOUR_IF_STAGE2_FAILS=2` doing its job. Check `docker logs`, and
set `-e S6_VERBOSITY=2` for a detailed trace.

**`[init-adduser] PUID invalide`**
`PUID` / `PGID` must be integers ≥ `1`; `0` is rejected because it is root's UID.

**Files created in a volume have the wrong owner**
Set `-e PUID=$(id -u) -e PGID=$(id -g)`.

More entries in the
[full documentation](https://github.com/Sam-Tech-Lab-Git/Docker-Ubuntu-18.04-Bionic-S6-Overlay-Test#troubleshooting).

---

## License

MIT — see
[LICENSE](https://github.com/Sam-Tech-Lab-Git/Docker-Ubuntu-18.04-Bionic-S6-Overlay-Test/blob/main/LICENSE).
Copyright (c) 2026 Sam Tech Lab.

---
---

# Ubuntu 18.04 LTS (Bionic) + s6-overlay — version française

Image de base **Ubuntu 18.04 LTS (Bionic)** minimale, durcie et multi-architecture, construite
`FROM scratch` à partir du rootfs OCI officiel d'Ubuntu, avec
[s6-overlay](https://github.com/just-containers/s6-overlay) comme système d'init et superviseur de
processus.

C'est une **fondation pour vos propres images** : elle fournit un système d'init, un utilisateur
non-root dont l'UID/GID est configurable à l'exécution, et un socle système durci — puis vous
laisse travailler.

**Documentation complète, en anglais et en français :**
<https://github.com/Sam-Tech-Lab-Git/Docker-Ubuntu-18.04-Bionic-S6-Overlay-Test>

---

## ⚠️ À lire d'abord — Ubuntu 18.04 est sorti du support standard

Ubuntu 18.04 LTS est sorti du support standard le **31 mai 2023**. Son archive publique ne reçoit
plus de nouvelles mises à jour de sécurité : celles-ci passent par Ubuntu Pro (ESM), auquel cette
image n'est pas abonnée.

- Les reconstructions mensuelles rafraîchissent l'image à partir de ce qu'Ubuntu sert encore pour
  Bionic. Elles n'apportent **pas** les correctifs qui n'existent que derrière l'ESM : une CVE
  corrigée pour 20.04 ou 22.04 peut rester ouverte ici indéfiniment.
- Les analyses de vulnérabilités publiées incluent volontairement les vulnérabilités **sans
  correctif disponible** — sur une archive figée, c'est la majorité d'entre elles. Les rapports
  sont longs, et c'est voulu.

Utilisez cette image là où un userland 18.04 est une contrainte dure : binaires hérités, ancienne
chaîne de compilation, reproduction d'un environnement historique. **Pour tout nouveau projet,
partez d'une LTS Ubuntu encore supportée.** Si vous devez exploiter 18.04 en production, ajoutez
un abonnement Ubuntu Pro dans votre image dérivée.

---

## Tags

| Tag | Contenu |
|---|---|
| `latest` | Suit la reconstruction mensuelle — amd64 + arm64 |
| `YYYY.MM` (par ex. `2026.08`) | Instantané mensuel immuable — amd64 + arm64 |

Les tags pointent vers un manifeste multi-architecture : Docker sélectionne l'image correspondant
à la plateforme hôte. Préférez `YYYY.MM` pour des déploiements reproductibles.

Également publiée sur GHCR : `ghcr.io/sam-tech-lab-git/ubuntu-18.04-bionic-s6`.

---

## Démarrage rapide

```bash
# Shell en tant qu'appuser, non privilégié
docker run -it --rm samtechlab/ubuntu-18.04-bionic-s6:latest

# Aligner l'utilisateur du conteneur sur celui de l'hôte
docker run --rm -e PUID=$(id -u) -e PGID=$(id -g) \
  -v "$PWD/data:/config" \
  samtechlab/ubuntu-18.04-bionic-s6:latest id appuser
```

Construire par-dessus — installez les paquets **au build**, en root :

```dockerfile
FROM samtechlab/ubuntu-18.04-bionic-s6:latest

RUN apt-get update && \
    apt-get install -y --no-install-recommends votre-paquet && \
    rm -rf /var/lib/apt/lists/*

COPY root/ /
```

> N'installez **pas** de paquets au démarrage du conteneur. Les services tournent sans privilèges,
> APT échoue donc avec `Permission denied` sur `/var/lib/apt/lists`.

---

## Déroulement du démarrage

```
docker run
   │
   ├─ 1. ENTRYPOINT /init      s6-overlay devient PID 1
   ├─ 2. oneshots s6-rc        init-adduser applique PUID/PGID   ← vos tâches d'init
   ├─ 3. longruns s6-rc        démarrage des daemons supervisés  ← vos services
   └─ 4. CMD                   le conteneur s'arrête avec lui
```

À l'arrêt, la séquence se déroule à l'envers : les services sont arrêtés dans l'ordre des
dépendances, puis les processus restants reçoivent `SIGTERM`, puis `SIGKILL` après un délai.

---

## Points forts

- Construite `FROM scratch` depuis le rootfs OCI officiel Ubuntu — aucune couche de base tierce
- **s6-overlay en PID 1** — nettoyage des zombies, démarrage et arrêt ordonnés, signaux corrects
- **`PUID` / `PGID` configurables à l'exécution** — appliqués avant tout service
- **Supervision multi-services** avec dépendances déclarées et redémarrage automatique
- Rotation des journaux par service via `logutil-service`
- **Non-root par défaut** — le `CMD` par défaut abandonne les privilèges vers `appuser`
- **Durcissement** — `root` verrouillé, bits SUID/SGID supprimés, bits world-writable retirés,
  `umask 027`
- **Intégrité de la chaîne d'approvisionnement** — builder Alpine figé par digest, tarballs
  s6-overlay figés par SHA256 et vérifiés avant extraction, actions CI figées par SHA de commit
- **Init fail-fast** — un script d'init en échec arrête le conteneur
- Vérifiée en continu — hadolint, shellcheck, 9 tests d'intégration, scans Trivy hebdomadaires

---

## Variables d'environnement

| Variable | Défaut | Description |
|---|---|---|
| `PUID` | `1000` | UID appliqué à `appuser` au démarrage |
| `PGID` | `1000` | GID appliqué à `appuser` au démarrage |
| `HOME` | `/config` | Répertoire personnel de `appuser` |
| `TZ` | `UTC` | Fuseau horaire |
| `LANG` | `en_US.UTF-8` | Locale (également `LANGUAGE`, `LC_ALL`) |
| `TERM` | `xterm` | Type de terminal |

`PUID` / `PGID` doivent être des entiers **≥ 1**. `0` est refusé volontairement : c'est l'UID de
root, et l'accepter ferait silencieusement d'`appuser` un second compte root. Toute valeur
invalide arrête le conteneur avec une erreur explicite.

Réglages s6-overlay définis par l'image, et ceux que vous pouvez définir vous-même :

| Variable | Valeur | Effet |
|---|---|---|
| `S6_BEHAVIOUR_IF_STAGE2_FAILS` | `2` | Arrête le conteneur si un script d'init échoue |
| `S6_CMD_WAIT_FOR_SERVICES_MAXTIME` | `0` | Aucun délai de démarrage imposé aux services |
| `S6_VERBOSITY` | `1` | Avertissements et erreurs — monter à `2`+ pour déboguer |
| `S6_SERVICES_GRACETIME` | `3000` | Millisecondes laissées aux services pour s'arrêter |
| `S6_KILL_GRACETIME` | `3000` | Millisecondes entre le `SIGTERM` final et le `SIGKILL` |
| `S6_READ_ONLY_ROOT` | `0` | Mettre à `1` avec une racine en lecture seule |

---

## Arborescence

| Chemin | Rôle |
|---|---|
| `/config` | Home de `appuser`, mode `750` — montez vos données persistantes ici |
| `/command` | Binaires s6 (`s6-setuidgid`, `with-contenv`, `s6-rc`, …) |
| `/etc/s6-overlay/s6-rc.d/` | Définitions de services |
| `/etc/s6-overlay/user-bundles.d/user/contents.d/` | Services activés au démarrage |
| `/etc/s6-overlay/scripts/` | Scripts shell appelés par les définitions de services |

---

## Ajouter un service supervisé

Gardez votre arborescence de services dans un répertoire `root/` du contexte de build, et copiez-la
en entier.

`root/etc/s6-overlay/s6-rc.d/myapp/type` → `longrun`

`root/etc/s6-overlay/s6-rc.d/myapp/run` — *doit être exécutable*
```sh
#!/command/with-contenv sh
exec 2>&1
# Le superviseur tourne en root ; le daemon ne doit pas.
exec s6-setuidgid appuser /usr/bin/myapp --foreground
```

`root/etc/s6-overlay/s6-rc.d/myapp/dependencies.d/init-adduser` — *fichier vide*
`root/etc/s6-overlay/user-bundles.d/user/contents.d/myapp` — *fichier vide*

```dockerfile
COPY root/ /
RUN chmod 755 /etc/s6-overlay/s6-rc.d/myapp/run
```

Deux règles importantes :

- **Le daemon doit rester au premier plan.** Un processus qui passe en arrière-plan ressemble à un
  crash pour le superviseur, qui le redémarre en boucle.
- **Abandonnez les privilèges avec `s6-setuidgid`.** Tout ce qui est sous `s6-rc.d` démarre en
  `root`.

L'ordonnancement entre services, la journalisation par service et un exemple NGINX complet et
fonctionnel sont couverts dans la
[documentation complète](https://github.com/Sam-Tech-Lab-Git/Docker-Ubuntu-18.04-Bionic-S6-Overlay-Test#ajouter-vos-propres-services).

---

## Modèle de sécurité

Le PID 1 tourne en `root` : s6-overlay a besoin de ces privilèges pour appliquer `PUID`/`PGID` et
céder la propriété à des processus non privilégiés. Un UID/GID configurable à l'exécution et un
PID 1 non-root sont mutuellement exclusifs. Tout ce qui se trouve au-dessus est conçu pour réduire
ce qui s'exécute réellement avec des privilèges.

| Contrôle | Mise en œuvre |
|---|---|
| `CMD` par défaut | S'exécute en `appuser`, pas en `root` |
| Compte `root` | Mot de passe verrouillé, `/root` en mode `700` |
| Shell de connexion d'`appuser` | `/usr/sbin/nologin` |
| Binaires SUID/SGID | Supprimés sur toute l'image au build |
| Fichiers world-writable | Bit d'écriture retiré sur toute l'image au build |
| Umask par défaut | `027` |
| `/config` | Mode `750`, appartenant à `appuser` |
| `PUID` / `PGID` | Validés au démarrage ; `0` refusé |
| Échec d'init | Arrête le conteneur |

**Ce qui reste à votre charge :** tout ce que vous ajoutez sous `s6-rc.d` démarre en `root`.
Encadrez chaque processus longue durée avec `s6-setuidgid appuser`.

Durcissement recommandé à l'exécution :

```yaml
services:
  app:
    image: samtechlab/ubuntu-18.04-bionic-s6:latest
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    tmpfs:
      - /tmp
```

Signalement de vulnérabilité :
[SECURITY.md](https://github.com/Sam-Tech-Lab-Git/Docker-Ubuntu-18.04-Bionic-S6-Overlay-Test/blob/main/SECURITY.md)

---

## Dépannage

**`E: Could not open lock file /var/lib/apt/lists/lock (13: Permission denied)`**
APT tourne sans privilèges. Installez les paquets au build, dans votre `Dockerfile`.

**`bind() to 0.0.0.0:80 failed (13: Permission denied)`**
Un processus non privilégié ne peut pas écouter sous le port 1024. Utilisez un port ≥ 1024 dans le
conteneur et remappez-le côté hôte (`-p 80:8080`).

**Un service redémarre en boucle**
Le processus passe en arrière-plan. Forcez le premier plan (`nginx -g "daemon off;"`,
`--foreground`, …).

**Le conteneur s'arrête immédiatement au démarrage**
Un script d'init a échoué — c'est `S6_BEHAVIOUR_IF_STAGE2_FAILS=2` qui joue son rôle. Consultez
`docker logs`, et passez `-e S6_VERBOSITY=2` pour une trace détaillée.

**`[init-adduser] PUID invalide`**
`PUID` / `PGID` doivent être des entiers ≥ `1` ; `0` est refusé car c'est l'UID de root.

**Les fichiers créés dans un volume ont le mauvais propriétaire**
Passez `-e PUID=$(id -u) -e PGID=$(id -g)`.

D'autres entrées dans la
[documentation complète](https://github.com/Sam-Tech-Lab-Git/Docker-Ubuntu-18.04-Bionic-S6-Overlay-Test#dépannage).

---

## Licence

MIT — voir
[LICENSE](https://github.com/Sam-Tech-Lab-Git/Docker-Ubuntu-18.04-Bionic-S6-Overlay-Test/blob/main/LICENSE).
Copyright (c) 2026 Sam Tech Lab.
