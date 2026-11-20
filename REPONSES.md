# TP 6 – Variables d'environnement, Inputs et Outputs : réponses

## Partie 1 – `GITHUB_ENV`

**Q1. Pourquoi la variable n'est-elle pas disponible dans la step qui l'a définie, mais seulement dans les suivantes ?**

`echo "APP_NAME=my-app" >> $GITHUB_ENV` n'exporte rien dans le shell en cours : la commande
écrit simplement une ligne dans un fichier temporaire dont le chemin est donné par `GITHUB_ENV`.
Le runner ne lit ce fichier **qu'à la fin de la step**, puis ajoute les variables à
l'environnement des **steps suivantes** au moment où il crée leur processus. Le processus bash
de la step courante a déjà été lancé avec son environnement : il ne voit pas la nouvelle
variable. Pour l'utiliser dans la même step, il faut aussi l'exporter dans le shell
(`export APP_NAME=my-app`).

**Q2. Job `verify-env` : que constate-t-on ? Quelle est la portée de `GITHUB_ENV` ?**

La step `Print APP_NAME` du job `verify-env` affiche `APP_NAME=` : la variable est **vide**
(il n'y a pas d'erreur, bash remplace simplement une variable inexistante par une chaîne vide).
Chaque job s'exécute sur sa propre machine virtuelle, avec son propre fichier `GITHUB_ENV` ;
`verify-env` tourne même en parallèle de `build`. La portée de `GITHUB_ENV` est donc **le job** :
la variable est visible dans les steps suivantes du même job, jamais dans un autre job.

## Partie 3 – Outputs entre jobs

**Q3. Que se passe-t-il avec `>` au lieu de `>>` dans `GITHUB_OUTPUT` si deux outputs sont définis dans la même step ?**

`>` **écrase** le fichier à chaque écriture, alors que `>>` **ajoute** une ligne à la fin.
Avec deux `echo ... > $GITHUB_OUTPUT` dans la même step, la seconde écriture efface la première :
à la fin de la step, le fichier ne contient que le dernier output. Le premier n'existe donc pas
et toute référence à `steps.<id>.outputs.<premier>` vaut une chaîne vide, sans message d'erreur.
Même chose avec `GITHUB_ENV`. Il faut toujours utiliser `>>`.

**Q4. Pourquoi `GITHUB_ENV` ne suffit-il pas ici, alors que `GITHUB_OUTPUT` fonctionne entre jobs ?**

`GITHUB_ENV` est un fichier local à la machine du job (voir Q2) : il disparaît avec la VM à la fin
du job `build`. `GITHUB_OUTPUT` sert à déclarer des outputs de step ; une fois **déclarés au niveau
du job** (`jobs.build.outputs.full_version`), GitHub les remonte et les conserve avec le run, puis
les met à disposition des jobs qui dépendent de `build` via le contexte
`needs.build.outputs.full_version`. C'est donc GitHub, pas la machine, qui transporte la valeur
d'un job à l'autre (ce mécanisme convient à des petites valeurs ; pour des fichiers, on utilise
les artefacts).

**Q5. Que se passe-t-il si on retire `needs: build` du job `deploy` ?**

- `deploy` ne dépend plus de `build` : les deux jobs démarrent **en parallèle**, `deploy`
  n'attend plus que `build` ait calculé la version, et il s'exécuterait même si `build` échouait.
- Le contexte `needs` de `deploy` ne contient plus `build` : `${{ needs.build.outputs.full_version }}`
  est évalué à une **chaîne vide**. Le job affiche `Deploying version ` sans valeur, sans erreur
  à l'exécution (un linter comme `actionlint` signale en revanche cette référence invalide).

`needs` sert donc à la fois à **ordonner** les jobs et à **donner accès** aux outputs du job dont
on dépend.

## Partie 4 – Workflow réutilisable

`greeting.yml` est déclenché uniquement par `workflow_call` et attend un input `recipient`
de type `string`. Le job `call-greeting` de `tp-env.yml` l'appelle avec
`uses: ./.github/workflows/greeting.yml` (pas de `runs-on` ni de `steps` dans le job appelant)
et lui transmet `${{ inputs.environment }}` : lancé avec `staging`, le job
`call-greeting / greet` affiche `Bonjour staging`.

## Exécution

Capture du run réussi (jobs `build`, `verify-env`, `deploy`, `call-greeting`) :
[`docs/tp6-run-success.jpg`](docs/tp6-run-success.jpg).
