# Rapport d'incident — Jour 5

## Incident — Mauvais tag d'image sur le déploiement `green`

**Contexte** : test volontaire d'un des scénarios cités par le brief (Phase 4), déclenché par moi-même pour m'entraîner au diagnostic avant la soutenance.

**Symptôme** : après un commit changeant le tag d'image de `kps-tasks-api-green` vers un tag inexistant (`v9.9.9-does-not-exist`) et une synchronisation ArgoCD, le nouveau pod `green` reste bloqué : `kubectl get pods` affiche `Init:ImagePullBackOff`.

**Diagnostic** :
```bash
kubectl describe pod kps-tasks-api-green-<hash> -n kps-tasks
```
La section `Events` donne la cause exacte, sans ambiguïté :
```
Failed to pull image "...:v9.9.9-does-not-exist": ... not found
```
Côté ArgoCD, `argocd app get kps-tasks-api-dev` confirmait `Sync Status: Synced` (le manifeste erroné a bien été appliqué tel quel — ArgoCD ne juge jamais si une configuration est correcte, voir question 46) mais `Health Status: Progressing`, qui serait passé à `Degraded` si l'état avait persisté.

**Impact réel** : nul. `blue` (la version active) n'a jamais été touché — l'application est restée disponible et fonctionnelle tout le long de l'incident, exactement le bénéfice attendu de la séparation blue/green (voir `docs/blue-green-deployment.md`).

**Correctif** :
```bash
# apps/kps-tasks-api/deployment-green.yaml : tag corrigé vers v1.2.0
git commit -am "fix: restore correct v1.2.0 image tag on green deployment"
git push
argocd app sync kps-tasks-api-dev
```
Vérifié : les trois pods (`blue`, `green`, `postgres`) `Running`, ArgoCD `Synced`/`Healthy`, `/health` répondant normalement.

**Prévention** : une revue avant fusion sur le dépôt GitOps (voir question 47) aurait permis de repérer un tag suspect avant qu'il n'atteigne le cluster. Une vérification automatique de l'existence du tag dans le registre, en amont du commit sur le dépôt GitOps (par exemple dans le pipeline qui met à jour ce tag), éliminerait cette classe d'erreur entièrement.

**Ce que ça illustre** : ArgoCD applique fidèlement ce que dit Git, sans jugement — la qualité du contenu du dépôt GitOps reste une responsabilité humaine, jamais déléguée à l'outil.

Preuve complète : `evidence/incident-bad-image-tag.txt`.
