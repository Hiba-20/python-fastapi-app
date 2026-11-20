# TP 7 – Sécurité des actions tierces

## Observations sur `actions/setup-node` (relevé du 20/11/2026)

| Élément | Constat |
|---|---|
| Propriétaire | Organisation `actions` (GitHub), badge **Verified creator** sur la Marketplace |
| Étoiles | ≈ 5 000 (5 028), ≈ 1 700 forks |
| Code source | TypeScript dans `src/` ; code compilé exécuté par le runner dans `dist/` (`action.yml` → `runs.using: node24`, `main: dist/setup/index.js`) |
| Autres repères | `LICENSE`, `CODEOWNERS`, tests dans `__tests__/`, dépôt actif (dernier push la veille) |

## Test `@main` puis retour à `@v4`

`actions/checkout@v4` a été remplacé par `actions/checkout@main` (commit dédié), le workflow
a tourné, puis la référence a été remise à `@v4` (commit suivant). Le run fonctionne dans les
deux cas, mais avec `@main` on exécute le code tel qu'il est sur la branche à cet instant,
sans aucune garantie de stabilité.

## Q3. Pourquoi épingler une version (`@v4`) plutôt que `@main` ?

- `main` est une branche : elle avance à chaque commit. Le workflow exécuterait du code
  différent d'un run à l'autre, potentiellement non publié, en cours de développement ou
  contenant une rupture de compatibilité (changement d'inputs, nouvelle version de Node…).
  Les builds ne sont plus reproductibles.
- Sécurité : si le dépôt de l'action est compromis, un commit malveillant poussé sur `main`
  serait exécuté immédiatement par tous les workflows qui pointent dessus, avec accès au
  `GITHUB_TOKEN` et aux secrets du job.
- `@v4` désigne une version majeure publiée : les mainteneurs n'y mettent que des correctifs
  compatibles. Encore mieux pour la production : épingler sur le **SHA complet du commit**
  (`actions/checkout@<sha40> # v4.x.y`), seule référence immuable — un tag peut être déplacé,
  comme l'a montré l'incident `tj-actions/changed-files` (mars 2025), où les tags ont été
  réécrits pour pointer vers du code qui exfiltrait les secrets.

## Q4. Le badge « créateur vérifié » garantit-il qu'une action est sûre ?

Non. Le badge certifie seulement **l'identité de l'éditeur** : GitHub a vérifié que le compte
appartient bien à l'organisation annoncée (ex. `actions`, `docker`, `aws-actions`). Il ne dit
rien sur le **code** : aucune revue de sécurité de chaque version, pas de garantie contre un
bug, une vulnérabilité, une dépendance compromise ou un compte de mainteneur piraté. Il réduit
le risque d'usurpation (typosquatting d'un faux « setup-node »), mais la confiance doit
reposer aussi sur l'analyse du code et l'épinglage de version.

## Q5. Deux vérifications avant d'utiliser en production une action d'un fournisseur inconnu

1. **Lire le code et le `action.yml`** de la version utilisée : ce qu'elle exécute
   (`dist/`, script, image Docker et son `Dockerfile`), les appels réseau, l'usage du token et
   des secrets, les dépendances. Puis **l'épingler par SHA de commit** pour exécuter exactement
   le code relu (ou la forker dans l'organisation).
2. **Évaluer la confiance dans le projet** : propriétaire et historique, activité et
   maintenance (commits récents, issues/PR traitées), popularité, licence, politique de
   sécurité (`SECURITY.md`, alertes Dependabot/advisories), score OpenSSF Scorecard.

Bonnes pratiques complémentaires : restreindre `permissions:` du `GITHUB_TOKEN` au strict
nécessaire, ne pas exposer de secrets aux jobs qui n'en ont pas besoin, activer Dependabot
pour suivre les mises à jour des actions, et limiter les actions autorisées dans les
paramètres du dépôt/de l'organisation.
