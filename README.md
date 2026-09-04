# kps-tasks-gitops

Dépôt GitOps pour KPS Tasks API — source de vérité pour l'état Kubernetes, synchronisée par [ArgoCD](https://argo-cd.readthedocs.io/). Projet 6 (OpsReady-06), Académie KPS Technologies.

Ce dépôt est volontairement séparé du dépôt applicatif ([`devops-prj3`](https://github.com/Ronaldo-F-dev/devops-prj3)), qui contient le code, le `Dockerfile` et le pipeline CI. Voir [`docs/gitops-principles.md`](docs/gitops-principles.md) pour la justification.

## Contenu

- `apps/kps-tasks-api/` — manifestes Kubernetes : `namespace.yaml`, ConfigMap/Secret d'exemple, `deployment-blue.yaml`/`deployment-green.yaml` (blue/green, voir Jour 4), `app-service.yaml` (sélecteur pilotant la version active), PostgreSQL (Deployment/Service/PVC/Secret d'exemple)
- `docs/` — documentation complète, jour par jour (voir index ci-dessous)
- `evidence/` — preuves capturées à chaque étape

## Règle d'or

Aucun `kubectl apply` manuel sur ce contenu une fois ArgoCD connecté — tout changement passe par un commit sur ce dépôt.

## Documentation

| Jour | Documents |
|---|---|
| 1 | [`docs/gitops-principles.md`](docs/gitops-principles.md), [`docs/argocd-installation.md`](docs/argocd-installation.md) |
| 2 | [`docs/argocd-application.md`](docs/argocd-application.md) |
| 3 | [`docs/sync-and-drift.md`](docs/sync-and-drift.md) |
| 4 | [`docs/blue-green-deployment.md`](docs/blue-green-deployment.md), [`docs/adr/ADR-002-blue-green-strategy.md`](docs/adr/ADR-002-blue-green-strategy.md) |
| 5 | [`docs/rollback-gitops.md`](docs/rollback-gitops.md), [`docs/incident-report.md`](docs/incident-report.md), [`docs/soutenance.md`](docs/soutenance.md) |
| — | [`docs/intermediate-questions.md`](docs/intermediate-questions.md) — toutes les questions 11-63, par jour |

## Application ArgoCD

- Nom : `kps-tasks-api-dev`
- Namespace cible : `kps-tasks`
- Interface : `https://<IP-VPS>:30843`
