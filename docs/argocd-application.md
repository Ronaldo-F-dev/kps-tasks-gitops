# Application ArgoCD et première synchronisation — Jour 2

## Un point critique traité avant de commencer : exclure les modèles de secrets

Le dossier `apps/kps-tasks-api/` contient, en plus des manifestes réels, deux fichiers `*.example.yaml` (modèles de `Secret`, avec des valeurs `REPLACE_ME`). Une source ArgoCD de type "répertoire" synchronise par défaut **tous** les fichiers `.yaml` du chemin donné — sans exclusion explicite, ArgoCD aurait tenté de créer un `Secret` avec des identifiants factices, écrasant les vrais secrets déjà en place sur le cluster. Traité avec l'option `--directory-exclude "*.example.yaml"` à la création de l'Application.

## Création de l'Application (tâches 17-21)

```bash
argocd app create kps-tasks-api-dev \
  --repo https://github.com/Ronaldo-F-dev/kps-tasks-gitops.git \
  --path apps/kps-tasks-api \
  --dest-server https://kubernetes.default.svc \
  --dest-namespace kps-tasks \
  --directory-exclude "*.example.yaml" \
  --sync-policy none
```

| Paramètre | Valeur | Pourquoi |
|---|---|---|
| Nom | `kps-tasks-api-dev` | Nom recommandé par le brief |
| Dépôt source | `kps-tasks-gitops` | Jamais le dépôt applicatif — c'est tout l'enjeu de la séparation (`docs/gitops-principles.md`) |
| Chemin | `apps/kps-tasks-api` | Le sous-dossier exact contenant les manifestes de cette application |
| Cluster cible | `https://kubernetes.default.svc` | Le cluster local — ArgoCD tourne dans le même cluster qu'il gère (déploiement le plus simple, un seul k3s) |
| Namespace cible | `kps-tasks` | Le même namespace créé au Projet 5 — ArgoCD en prend la gestion, sans le recréer différemment |
| Politique de synchronisation | `none` (manuelle) | Choix volontaire du brief : comprendre le mécanisme manuel avant d'activer l'automatisation (Jour 3) |

## Première synchronisation (tâche 22)

```bash
argocd app sync kps-tasks-api-dev
```

**Ce que cette première synchronisation a réellement fait** : le cluster faisait encore tourner l'image `commit-ed490e5` (dernier état laissé par le Projet 5), alors que le dépôt GitOps référence déjà `v1.1.0` (voir `docs/prj6/overview.md` du dépôt applicatif pour le contexte du tag). La synchronisation n'était donc pas une simple formalité : elle a déclenché un vrai rollout Kubernetes, sans qu'aucune commande `kubectl` n'ait été tapée pour ça — seul `argocd app sync` a suffi.

## Vérification (tâches 23-25)

```bash
argocd app get kps-tasks-api-dev
```

Résultat : `Sync Status: Synced`, `Health Status: Healthy`, les sept objets attendus (Namespace, ConfigMap, PVC, 2 Services, 2 Deployments) tous `Synced`. Confirmation applicative :

```bash
curl http://<IP-VPS>:30080/version
# {"name":"KPS Tasks API","version":"1.1.0","environment":"production"}
```

La version affichée correspond exactement à celle du dépôt GitOps — preuve que le déploiement est désormais entièrement piloté par Git. Preuve complète : `evidence/app-synced-healthy.txt`.

## Schéma du flux

```
Commit sur kps-tasks-gitops (déjà fait au Jour 1 : image v1.1.0)
  → argocd app sync (déclenché manuellement)
  → ArgoCD compare l'état désiré (Git) à l'état réel (cluster)
  → ArgoCD applique les objets manquants/différents (kubectl apply équivalent, exécuté par ArgoCD lui-même)
  → Kubernetes effectue le rollout du Deployment
  → Application accessible, /version reflète le contenu de Git
```
