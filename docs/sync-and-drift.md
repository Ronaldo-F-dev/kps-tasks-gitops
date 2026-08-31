# Changement de version, synchronisation automatisée et drift — Jour 3

## Changement de version depuis Git (tâches 32-37)

Une nouvelle image `v1.2.0` a d'abord été construite côté dépôt applicatif (nouveau tag Git `v1.2.0`, pipeline CI existant, sans aucun changement de processus par rapport au Projet 4). Seule l'étape suivante concerne le GitOps :

```bash
# Dans kps-tasks-gitops
sed -i 's#v1.1.0#v1.2.0#g' apps/kps-tasks-api/app-deployment.yaml
git commit -am "chore: bump kps-tasks-api to v1.2.0"
git push
```

Résultat observé, sans aucune commande Kubernetes :

1. `argocd app get kps-tasks-api-dev` passe de `Synced` à `OutOfSync from (c5b0b44)` — le hash du nouveau commit apparaît immédiatement.
2. Seul l'objet réellement concerné (`Deployment kps-tasks-api`) est marqué `OutOfSync` — tous les autres objets restent `Synced`, preuve qu'ArgoCD compare précisément chaque ressource, pas l'application dans son ensemble en bloc.
3. `argocd app sync kps-tasks-api-dev` déclenche le rollout Kubernetes.
4. `curl .../version` confirme `"version":"1.2.0"`.

Preuve : `evidence/outofsync-detected.txt`.

## Synchronisation automatisée (tâche 38)

```bash
argocd app set kps-tasks-api-dev --sync-policy automated
```

**Décision volontaire prise à ce stade : activer *automated* sans activer *self-heal*.** Ce sont deux réglages distincts, souvent confondus :

| Réglage | Ce qu'il déclenche |
|---|---|
| `automated` (seul) | Dès que Git change, ArgoCD synchronise tout seul — plus besoin de taper `argocd app sync`. Ne surveille **pas** activement les changements faits directement dans le cluster. |
| `automated` + `self-heal` | En plus de ce qui précède, ArgoCD **annule automatiquement** toute modification faite directement dans le cluster, dès qu'il la détecte — sans attendre qu'un humain lance une synchronisation manuelle. |

Ce choix a été fait exprès pour pouvoir observer la tâche suivante (le drift) sans qu'ArgoCD ne le corrige tout seul avant qu'on ait pu l'examiner.

## Provoquer et corriger un drift (tâches 39-41)

**Provoqué** avec une commande `kubectl` directe, sans toucher à Git :

```bash
kubectl scale deployment/kps-tasks-api -n kps-tasks --replicas=2
```

**Détecté** par ArgoCD, exactement de la même façon qu'un changement de version au bloc précédent — `OutOfSync`. C'est le point le plus important de ce module : **pour ArgoCD, "Git a changé" et "le cluster a changé" produisent le même signal.** Il n'y a pas de distinction entre "écart voulu, pas encore appliqué" et "écart non voulu, fait en douce" — dans les deux cas, c'est un désaccord entre état désiré et état réel.

```bash
argocd app diff kps-tasks-api-dev
```
```
===== apps/Deployment kps-tasks/kps-tasks-api ======
201c201
<   replicas: 2
---
>   replicas: 1
```
(`<` = état réel du cluster, `>` = état désiré dans Git)

**Corrigé** en resynchronisant — ce qui, dans ce sens-là, revient à *annuler* le changement manuel plutôt qu'à en appliquer un nouveau :

```bash
argocd app sync kps-tasks-api-dev
```

Résultat : retour à 1 seul pod, `Synced`, `Healthy`. Preuves : `evidence/drift-detected.txt`, `evidence/drift-corrected.txt`.

## Pourquoi ce n'était pas automatique (rappel du choix du bloc précédent)

Parce que `self-heal` n'a volontairement pas été activé. Si ça avait été le cas, ArgoCD aurait annulé le `kubectl scale` de lui-même, en quelques secondes, sans qu'on ait eu besoin de taper `argocd app sync` — la correction aurait été aussi automatique que la détection. Les deux comportements sont légitimes ; celui choisi ici sert uniquement à bien séparer "détecter" de "corriger" pour les besoins de la démonstration.
