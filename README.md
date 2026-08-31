# kps-tasks-gitops

Dépôt GitOps pour KPS Tasks API — source de vérité pour l'état Kubernetes, synchronisée par [ArgoCD](https://argo-cd.readthedocs.io/). Projet 6 (OpsReady-06), Académie KPS Technologies.

Ce dépôt est volontairement séparé du dépôt applicatif ([`devops-prj3`](https://github.com/Ronaldo-F-dev/devops-prj3)), qui contient le code, le `Dockerfile` et le pipeline CI. Voir [`docs/gitops-principles.md`](docs/gitops-principles.md) pour la justification.

## Contenu

- `apps/kps-tasks-api/` — manifestes Kubernetes de base (Namespace, Deployment, Service, ConfigMap, Secret d'exemple, PostgreSQL)
- `environments/dev/` — surcouches spécifiques à l'environnement (à venir)
- `docs/` — documentation du fonctionnement GitOps, de l'installation ArgoCD, de la stratégie blue/green
- `evidence/` — preuves capturées (pods, synchronisation, drift, bascule blue/green)

## Règle d'or

Aucun `kubectl apply` manuel sur ce contenu une fois ArgoCD connecté — tout changement passe par un commit sur ce dépôt.
