# Déploiement blue/green — Jour 4

## Mécanisme choisi (tâches 48-51)

Option A du brief (obligatoire) : **deux `Deployment` distincts, même namespace**, plutôt que deux namespaces séparés.

- `deployment-blue.yaml` → `kps-tasks-api-blue`, image `v1.1.0`, labels `app: kps-tasks-api, version: blue`
- `deployment-green.yaml` → `kps-tasks-api-green`, image `v1.2.0`, labels `app: kps-tasks-api, version: green`
- `app-service.yaml` → `selector: {app: kps-tasks-api, version: <couleur active>}`

Les deux Deployments tournent **en permanence**, tous les deux `Running`/`Healthy` — c'est le `selector` du Service, et lui seul, qui décide laquelle des deux versions reçoit réellement le trafic. Basculer, c'est changer une seule ligne (`version: blue` → `version: green`) dans un seul fichier.

## Basculement (tâches 52-54)

```bash
sed -i 's/version: blue/version: green/' apps/kps-tasks-api/app-service.yaml
git commit -am "chore: switch traffic from blue to green" && git push
argocd app sync kps-tasks-api-dev
```

Vérification : `curl /version` passe de `1.1.0` à `1.2.0`, et `kubectl get endpoints kps-tasks-api` ne montre plus qu'une seule IP — celle du pod `green`. Aucune coupure observée : les deux pods étaient déjà `Running` et `Ready` avant le changement, la bascule est donc instantanée (pas de temps de démarrage à attendre, contrairement à un rollout classique). Retour à `blue` effectué de la même façon, en sens inverse — preuve que l'opération est réversible aussi facilement dans les deux sens.

Preuves : `evidence/blue-active.txt`, `evidence/green-active.txt`.

## Ce que le sélecteur fait exactement (question 60)

Un `Service` Kubernetes ne route jamais vers un `Deployment` par son nom — il route vers **n'importe quel pod dont les labels correspondent à son `selector`**, peu importe de quel Deployment ce pod provient. En donnant aux pods `blue` et `green` le même label `app: kps-tasks-api` mais un label `version` différent, le Service peut cibler l'un ou l'autre en ne changeant qu'une seule valeur — sans jamais toucher aux Deployments eux-mêmes.

## Voir aussi

- `docs/adr/ADR-002-blue-green-strategy.md` — décision, alternatives, risques
- `docs/intermediate-questions.md`, questions 58-63
