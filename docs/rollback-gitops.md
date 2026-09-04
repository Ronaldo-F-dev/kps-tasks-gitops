# Procédure de rollback GitOps

Trois façons différentes de "revenir en arrière" existent sur ce projet, à ne pas confondre — chacune répond à un problème différent.

## 1. Rollback Git (annuler un mauvais commit sur le dépôt GitOps)

C'est la méthode à privilégier dans un vrai modèle GitOps : le dépôt Git étant la source de vérité, revenir en arrière signifie littéralement revenir à un état antérieur de Git.

```bash
git revert <commit-a-annuler>
git push origin main
argocd app sync kps-tasks-api-dev   # ou automatique si le sync policy l'est
```

`git revert` (plutôt que `git reset`) crée un nouveau commit qui annule le précédent — l'historique reste complet et lisible, contrairement à une réécriture d'historique qui effacerait la trace du problème.

## 2. Rollback Kubernetes natif (`kubectl rollout undo`)

Existe indépendamment de GitOps (vu au Projet 5) — revient à la révision précédente d'un `Deployment` directement au niveau du cluster. **À éviter une fois ArgoCD en place** : ce type de rollback est un `kubectl` direct sur le cluster, donc un drift par définition (voir `docs/sync-and-drift.md`). ArgoCD le détecterait et, en mode automatisé avec `self-heal`, l'annulerait purement et simplement.

## 3. Rollback blue/green (bascule de Service)

Le plus rapide des trois : remettre le `selector` du Service sur la couleur précédente (voir `docs/blue-green-deployment.md`). Contrairement aux deux autres, il ne s'agit pas de "défaire" un déploiement — les deux versions tournent déjà, prêtes ; on change juste la destination du trafic.

## Comparatif

| Méthode | Ce qui revient en arrière | Vitesse | À utiliser quand |
|---|---|---|---|
| Rollback Git | Le contenu du dépôt GitOps | Le temps d'un sync (secondes) | Un changement de configuration ou de version s'avère mauvais |
| Rollback Kubernetes natif | L'état du Deployment dans le cluster | Le temps d'un rollout | Jamais en présence d'ArgoCD — contourne la source de vérité |
| Rollback blue/green | La version qui reçoit le trafic | Quasi instantané | Une nouvelle version démarre correctement mais se comporte mal sous trafic réel |

Dans les trois cas, le principe reste le même que celui du Jour 3 : la correction doit passer par Git (rollback Git ou blue/green piloté depuis Git), jamais par une action manuelle isolée sur le cluster.
