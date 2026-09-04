# ADR-002 : Stratégie de déploiement blue/green

## Statut
Acceptée

## Contexte

Le client veut pouvoir changer de version en production sans coupure de service perceptible, et pouvoir revenir en arrière instantanément si la nouvelle version pose problème. Le rollout Kubernetes classique (Projet 5) remplace progressivement les pods d'une même version — pendant la transition, il n'existe jamais deux versions complètes et indépendantes disponibles simultanément.

## Options envisagées

| Option | Description | Avantage | Limite |
|---|---|---|---|
| A — Deux Deployments, même namespace | `kps-tasks-api-blue` et `kps-tasks-api-green`, un Service dont le sélecteur choisit lequel est actif | Simple à comprendre, bascule quasi instantanée (juste un label à changer) | Demande de bien maîtriser labels/sélecteurs — une erreur de label routerait vers rien ou vers les deux |
| B — Deux namespaces séparés | `app-blue` / `app-green`, chacun avec sa propre copie complète des objets | Isolation plus propre, aucun risque de collision de labels | Plus lourd à maintenir (tout dupliqué : ConfigMap, Secret, Service), bascule plus complexe (changer l'Ingress/DNS externe plutôt qu'un simple selector) |

## Décision

Option A retenue, conformément à la recommandation du brief : deux `Deployment` dans le namespace `kps-tasks`, un seul `Service` dont le `selector` inclut un label `version` (`blue` ou `green`).

## Justification

- Meilleur rapport simplicité/valeur pédagogique pour une première expérience blue/green.
- La bascule se résume à **un seul changement dans un seul fichier Git** (`app-service.yaml`), ce qui rend le mécanisme facile à auditer et à expliquer.
- Les deux versions partagent la même base de données PostgreSQL et la même configuration (ConfigMap/Secret) — pas de duplication inutile pour ce qui n'a pas besoin de varier entre les deux couleurs.

## Risques

- **Partage de la base de données** (voir question 62) : si `green` introduisait une migration de schéma incompatible avec `blue`, les deux versions ne pourraient plus cohabiter en toute sécurité — ce risque n'existe pas dans ce projet (aucun changement de schéma entre `v1.1.0` et `v1.2.0`), mais devrait être traité explicitement avant tout vrai changement de structure de données.
- **Erreur de label** : un `selector` mal orthographié routerait silencieusement vers zéro pod (`Service` sans endpoints) plutôt que de signaler une erreur explicite.
- **Consommation de ressources doublée** : les deux versions tournent en permanence, même quand une seule sert du trafic — acceptable ici (charge très faible), à surveiller à plus grande échelle.

## Limites

- Bascule **manuelle** — aucune automatisation ne décide quand basculer (voir question 63).
- Aucun test de santé de `green` avant bascule au-delà de son état `Healthy` général (pas de vérification applicative ciblée avant d'y envoyer du trafic réel).
- Pas de bascule progressive (10 % du trafic, puis 50 %, puis 100 %) — le changement est total et instantané.

## Amélioration possible

Un Ingress avec une règle de poids (canary) permettrait une bascule progressive plutôt que totale — hors périmètre de ce projet, mentionné comme piste pour une observabilité plus avancée (voir Jour 5).
