# TP 7 – Actions tierces et actions personnalisées : réponses

## Partie 1 – Actions tierces

**Q1. Pourquoi `actions/checkout` n'a-t-il pas besoin de `with` alors que `setup-node` en a besoin ?**

Tous les inputs de `actions/checkout` ont une valeur par défaut adaptée au cas courant :
`repository` vaut le dépôt courant (`${{ github.repository }}`), `ref` le commit qui a déclenché
le workflow, `token` le `GITHUB_TOKEN`, `path` le workspace. Sans paramètre, il fait donc déjà
ce qu'on attend : cloner le dépôt courant au bon commit.

`actions/setup-node` n'a pas de version par défaut « évidente » : sans `node-version`
(ou `node-version-file`), il n'installe rien et laisse le Node préinstallé du runner, dont la
version peut changer à chaque mise à jour de l'image. On passe donc `node-version: 20` dans
`with` pour obtenir un environnement explicite et reproductible.

**Q2. Pourquoi le job `verify` ne retrouve-t-il pas `report.txt` sans l'artefact ?**

Chaque job s'exécute sur sa propre machine virtuelle, neuve et isolée. `report.txt` est créé
pendant l'exécution du job `build`, dans le workspace de sa VM ; il n'est jamais commité dans
le dépôt. Quand `build` se termine, sa VM est détruite avec le fichier. Partager le même dépôt
ne partage que le code versionné (et `verify` ne fait même pas de checkout). Pour transmettre
un fichier produit d'un job à l'autre, il faut le publier avec `actions/upload-artifact` puis
le récupérer avec `actions/download-artifact` ; `needs: build` garantit l'ordre.

## Partie 3 – Action composite

**Q6. Pourquoi `actions/checkout` doit-il s'exécuter avant l'appel d'une action locale ?**

Une action locale est référencée par un chemin dans le dépôt (`uses: ./.github/actions/build-info`).
Au démarrage du job, le workspace du runner est vide : les fichiers du dépôt, dont
`action.yml`, n'y sont présents qu'après `actions/checkout`. Sans checkout, le runner échoue
avec une erreur du type « Can't find 'action.yml' ». À l'inverse, une action distante
(`owner/repo@v4`) est téléchargée automatiquement par le runner.

**Q7. Pourquoi `shell: bash` est-il obligatoire sur les steps `run` d'une action composite ?**

Dans un workflow, le shell d'une step `run` a une valeur par défaut (bash sur Linux/macOS,
pwsh sur Windows, ou `defaults.run.shell`). Une action composite, elle, est réutilisable
depuis n'importe quel workflow et n'importe quel OS : elle n'hérite pas de ces valeurs par
défaut. GitHub impose donc que chaque step `run` déclare son shell explicitement, ce qui
rend le comportement de l'action identique quel que soit l'appelant ; sans `shell`, le
workflow est rejeté à la validation.

## Remarque sur le déclencheur

En plus de `workflow_dispatch`, le workflow a un déclencheur `push` sur `feature/tp-actions` :
le bouton « Run workflow » n'apparaît que pour un workflow présent sur la branche par défaut
(`main`), ce qui n'est pas le cas ici. Le `push` permet de l'exécuter depuis la branche du TP.

## Partie 4 – Action Docker : comparaison des temps d'exécution

Mesures sur le run [#3](https://github.com/Hiba-20/python-fastapi-app/actions/runs/44760752411)
(durées arrondies à la seconde par l'API GitHub) :

| Action | Type | Durée de la step |
|---|---|---|
| `Run build info action` (`build-info`) | composite | < 1 s (0 s) |
| `Run docker hello action` (`docker-hello`) | Docker (`image: Dockerfile`) | ≈ 2 s |

L'action composite est quasi instantanée : ses steps s'exécutent directement dans le shell du
runner, sans rien préparer. L'action Docker est plus lente car, à chaque exécution, le runner
doit **construire l'image** à partir du `Dockerfile` (téléchargement de `alpine:3.20`, `COPY`,
`chmod`), puis **démarrer un conteneur** pour exécuter `entrypoint.sh`. Sur une image plus
lourde (dépendances installées dans le Dockerfile), l'écart se compte en dizaines de secondes.
Autres limites : une action Docker ne tourne que sur des runners Linux, alors qu'une composite
fonctionne sur tous les OS. Pour réduire le coût, on peut pointer `image:` vers une image
pré-construite (`docker://ghcr.io/...`) plutôt qu'un Dockerfile.

Capture du run réussi avec l'artefact `build-report` : [`docs/tp7-run-success.jpg`](docs/tp7-run-success.jpg).
