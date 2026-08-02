# Security Policy — Docker Ubuntu 18.04 LTS (Bionic)

This repository publishes a **minimal, hardened Ubuntu 18.04 (Bionic) base image** for container use.

---

## Supported Versions

| Version | Status |
|---|---|
| `latest` | ✅ Supported |
| Latest monthly tag (`YYYY.MM`) | ✅ Supported |
| Older tags | ❌ Not supported |

Only the most recent published tags receive updates and security follow-up.

---

## Security Baseline

The image includes the following default protections:

- non-root runtime user: `appuser`
- locked `root` account
- restrictive `umask 027`
- reduced SUID/SGID exposure
- blocked automatic service start during package installation
- cleaned APT caches, temp files, and logs during build
- official Ubuntu OCI rootfs as the base source

---

## Vulnerability Scanning

Security scanning is automated with [Trivy](https://github.com/aquasecurity/trivy).

### Reports are published to

| Location | Format | Access |
|---|---|---|
| GitHub **Security → Code scanning** | SARIF | Repository security tab |
| GitHub Actions step summary | Markdown | Workflow run summary |
| GitHub Actions artifacts | JSON | Downloadable artifact |

The scan workflow runs:
- every week on Monday at **04:00 UTC**
- automatically after successful image build workflows
- manually through GitHub Actions if needed

---

## Reporting a Vulnerability

If you discover a security issue in this repository, please **do not open a public issue**.

Use GitHub private vulnerability reporting instead:

1. Open the repository **[Security tab](https://github.com/Sam-Tech-Lab-Git/Docker-Ubuntu-18.04-Bionic/security)**
2. Click **Report a vulnerability**
3. Provide:
   - a clear description
   - reproduction steps
   - impact and affected area

### Response targets

- acknowledgment within **5 business days**
- mitigation or remediation target within **30 days**, depending on severity

---

## Scope

### In scope
- Dockerfile misconfigurations
- privilege escalation risks
- exposed secrets or CI/CD injection issues
- vulnerable packages present in the published image

### Out of scope
- upstream Ubuntu issues with no available fix yet
- vulnerabilities in third-party images built on top of this image

---

## Version française

## Versions supportées

| Version | Statut |
|---|---|
| `latest` | ✅ Supportée |
| Dernier tag mensuel (`YYYY.MM`) | ✅ Supporté |
| Anciens tags | ❌ Non supportés |

Seules les versions les plus récentes publiées reçoivent un suivi de sécurité.

## Mesures de sécurité

L’image applique notamment :
- un utilisateur non-root par défaut : `appuser`
- le verrouillage du compte `root`
- un `umask 027`
- la réduction des bits SUID/SGID inutiles
- le blocage du démarrage automatique des services à l’installation
- le nettoyage des caches APT, fichiers temporaires et journaux

## Analyse des vulnérabilités

L'analyse de sécurité est automatisée avec [Trivy](https://github.com/aquasecurity/trivy).

### Les rapports sont publiés dans

| Emplacement | Format | Accès |
|---|---|---|
| GitHub **Security → Code scanning** | SARIF | Onglet Security du dépôt |
| Résumé d'étape GitHub Actions | Markdown | Résumé du run de workflow |
| Artefacts GitHub Actions | JSON | Artefact téléchargeable |

Le workflow d'analyse s'exécute :
- chaque semaine, le lundi à **04h00 UTC**
- automatiquement après le succès des workflows de build de l'image
- manuellement via GitHub Actions si besoin

## Signalement d’une vulnérabilité

Merci de **ne pas ouvrir d’issue publique**.

Utilisez le signalement privé GitHub :
1. ouvrir l’onglet **[Security](https://github.com/Sam-Tech-Lab-Git/Docker-Ubuntu-18.04-Bionic/security)**
2. cliquer sur **Report a vulnerability**
3. décrire le problème, les étapes de reproduction et l’impact

Objectifs de réponse :
- accusé de réception sous **5 jours ouvrés**
- correction ou mitigation visée sous **30 jours** selon la sévérité

## Périmètre

### Dans le périmètre
- mauvaises configurations du Dockerfile
- risques d'élévation de privilèges
- secrets exposés ou problèmes d'injection CI/CD
- paquets vulnérables présents dans l'image publiée

### Hors périmètre
- problèmes Ubuntu amont sans correctif disponible pour le moment
- vulnérabilités dans des images tierces construites à partir de cette image

---

## License / Licence

This project is distributed under the **MIT** license. See [`LICENSE`](./LICENSE).

Ce projet est distribué sous la licence **MIT**. Voir [`LICENSE`](./LICENSE).

```text
Copyright (c) 2026 Sam Tech Lab
```
