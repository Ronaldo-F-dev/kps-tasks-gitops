# Support de soutenance — Projet 6 : OpsReady-06

Support pour une présentation de 10 à 15 minutes. Chaque section correspond à un point attendu du brief.

## 1. Contexte client

LogiCare Solutions pilotait Kubernetes à la main (`kubectl apply`, Projet 5) : personne n'était jamais sûr de la version réellement déployée, des changements manuels finissaient par diverger de ce qui était documenté, les rollbacks restaient improvisés. Objectif : faire de Git la source de vérité du cluster, via ArgoCD.

## 2. Principe du GitOps

Un modèle **pull** plutôt que **push** : au lieu qu'un pipeline pousse activement des changements vers le cluster, un contrôleur (ArgoCD) tourne dans le cluster, compare en continu l'état désiré (Git) à l'état réel, et corrige l'écart lui-même. Détail : `docs/gitops-principles.md`.

## 3. Architecture mise en place

```
Développeur → devops-prj3 (code) → pipeline CI → image taguée sur GHCR
                                                        │
                                    (mise à jour manuelle du tag)
                                                        ▼
                                          kps-tasks-gitops (manifestes)
                                                        │
                                              ArgoCD détecte, compare
                                                        ▼
                                        Cluster k3s (namespace kps-tasks)
```

## 4. Séparation des dépôts

`devops-prj3` (code, CI, Dockerfile) et `kps-tasks-gitops` (manifestes, environnements, blue/green) — rythmes de changement et niveaux de protection différents. Détail : `docs/gitops-principles.md`, question 12.

## 5. Installation ArgoCD

Sept pods dans le namespace `argocd`. Incident réel rencontré et corrigé : les CRD trop volumineuses pour `kubectl apply` classique (résolu avec `--server-side`), et un mot de passe admin exposé par erreur dans un terminal partagé (changé immédiatement, procédure documentée). Détail : `docs/argocd-installation.md`.

## 6. Application ArgoCD

`kps-tasks-api-dev` : dépôt, chemin `apps/kps-tasks-api`, cluster local, namespace `kps-tasks`. Piège évité : exclusion explicite des fichiers `*.example.yaml` (modèles de secrets) pour qu'ArgoCD ne les synchronise jamais. Détail : `docs/argocd-application.md`.

## 7. Fonctionnement de la synchronisation

Manuelle d'abord (`argocd app sync`), puis automatisée (`--sync-policy automated`) — en distinguant bien "automated" (applique les changements Git tout seul) de "self-heal" (corrige aussi un drift tout seul, volontairement laissé désactivé pour ce projet). Détail : `docs/sync-and-drift.md`.

## 8. Le concept de drift

Une modification faite directement dans le cluster (`kubectl scale`), invisible pour quiconque ne regarde pas ArgoCD activement. Détecté (`OutOfSync`, diff exact via `argocd app diff`), corrigé en revenant à ce que dit Git. Le point central : ArgoCD ne distingue pas "Git a changé" de "le cluster a changé" — les deux produisent le même signal.

## 9. Le mécanisme blue/green

Deux `Deployment` (`kps-tasks-api-blue` v1.1.0, `kps-tasks-api-green` v1.2.0) tournant en permanence dans le même namespace ; un seul `Service` dont le `selector` (label `version`) décide qui reçoit le trafic. Bascule et retour testés, sans interruption observée. ADR : `docs/adr/ADR-002-blue-green-strategy.md`.

## 10. Incident diagnostiqué

Tag d'image inexistant sur `green`, déclenché volontairement : `Init:ImagePullBackOff`, diagnostiqué via `kubectl describe` (message d'erreur explicite dans `Events`), corrigé en une commande Git. Aucun impact sur `blue`, actif pendant tout l'incident. Détail : `docs/incident-report.md`.

## 11. Limites du modèle actuel

- Synchronisation `automated` sans `self-heal` : un drift doit encore être corrigé manuellement (choix volontaire de ce projet, pas une contrainte technique)
- Blue/green manuel, sans bascule progressive ni critère automatique de décision
- Un seul nœud Kubernetes, une seule instance PostgreSQL — aucune haute disponibilité réelle
- Pas de politique de verrouillage de branche configurée sur le dépôt GitOps (mentionné comme risque, non mis en œuvre)

## 12. Améliorations possibles avant une observabilité complète

- Activer `self-heal` une fois la confiance dans le mécanisme établie
- Introduire un Ingress avec bascule progressive (canary) plutôt qu'un blue/green total
- Ajouter des métriques et des alertes (Prometheus/Grafana) pour objectiver une décision de bascule plutôt que de la faire à l'œil
- Protéger la branche `main` du dépôt GitOps par une revue obligatoire

## Questions probables

Toutes les réponses détaillées (11-63) sont dans `docs/intermediate-questions.md`, organisées par jour.
