# Synchronisation, changement de version et drift — Jour 3

## Changement de version depuis Git (tâches 32-37)

Un nouveau tag `v1.2.0` a été créé sur le dépôt applicatif (build/push automatique via le pipeline CI existant), puis **seul le dépôt GitOps a été modifié** — `apps/kps-tasks-api/app-deployment.yaml`, ligne `image:` et variable `APP_VERSION`, `v1.1.0` → `v1.2.0`. Commit et push sur `kps-tasks-gitops`.

ArgoCD a détecté l'écart automatiquement (`Sync Status: OutOfSync from (c5b0b44)`) — seul l'objet `Deployment kps-tasks-api` est passé `OutOfSync`, tout le reste (ConfigMap, Services, PVC, Deployment postgres) est resté `Synced`, preuve qu'ArgoCD ne réévalue pas tout en bloc mais compare objet par objet.

Synchronisation manuelle (`argocd app sync kps-tasks-api-dev`) : rollout Kubernetes déclenché, nouveau pod `Running`, `curl /version` confirmé à `1.2.0`. Preuves : `evidence/outofsync-detected.txt`, logs de synchronisation.

## Synchronisation automatisée (tâche 38)

```bash
argocd app set kps-tasks-api-dev --sync-policy automated
```

Décision volontaire : **auto-sync activé, `self-heal` laissé désactivé**, pour pouvoir observer séparément la détection et la correction d'un drift (voir plus bas). `--sync-policy automated` seul ne fait qu'une chose : appliquer sans attendre de confirmation humaine un changement **venant de Git**. Il ne surveille pas ce qui se passe dans le cluster en dehors de Git — c'est précisément le rôle de `self-heal` (voir question 42).

## Drift volontaire (tâches 39-41)

```bash
kubectl scale deployment/kps-tasks-api -n kps-tasks --replicas=2
```

Résultat : deux pods au lieu d'un, alors que le dépôt GitOps dit toujours `replicas: 1`. ArgoCD a signalé `OutOfSync`, avec un diff exact et lisible :

```bash
argocd app diff kps-tasks-api-dev
# < replicas: 2   (état réel)
# > replicas: 1   (état désiré, Git)
```

**Correction** : `argocd app sync kps-tasks-api-dev` a ramené le cluster à 1 réplica, conforme à Git — pas l'inverse. Preuve : `evidence/drift-detected.txt`.

## Push vs pull, avec ce qu'on vient de vivre comme exemple

| | Push (`kubectl apply`, Projets 1-5) | Pull (GitOps, ce projet) |
|---|---|---|
| Qui déclenche | Un humain ou un pipeline, depuis l'extérieur | ArgoCD, depuis l'intérieur du cluster |
| Ce qui s'est passé ici | Le `kubectl scale` du drift était un acte "push" — imposé au cluster sans passer par Git | La correction (`argocd app sync`) est un acte "pull" — ArgoCD est allé chercher l'état désiré dans Git et l'a réappliqué |

Le drift de ce Jour 3 est en réalité une démonstration de ce qui se passe quand on retombe, même une seule fois, dans le réflexe "push" — et pourquoi le modèle pull le rattrape.

Détail complet des réponses conceptuelles ci-dessous.
