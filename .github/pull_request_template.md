## Ce qui change, et pourquoi

<!--
Le « pourquoi » compte davantage que le « quoi » : le diff dit déjà quoi.
Si la modification corrige un comportement, décrivez ce qui se produisait avant.
-->

## Vérifications

<!-- Les commandes sont détaillées dans CONTRIBUTING.md. -->

- [ ] `hadolint` passe sur `Dockerfile-multi-arch`
- [ ] `shellcheck -s sh root/etc/s6-overlay/scripts/*` passe
- [ ] L'image se construit et démarre localement
- [ ] Documentation bilingue mise à jour si le comportement observable change
- [ ] `README.md` et `README-dockerhub.md` restent d'accord sur les faits

<!--
Rappels utiles :
- la CI exécute les tests d'intégration sur amd64 ET arm64 ; la publication
  attend les deux ;
- aucune image n'est publiée depuis une pull request ;
- si vous montez la version de s6-overlay, les quatre valeurs (version et
  trois empreintes) doivent être mises à jour ensemble, plus le badge du
  README.
-->
