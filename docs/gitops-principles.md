# Principes du GitOps

## Ce que le GitOps change

Jusqu'au Projet 5, le déploiement suivait un modèle **push** : un humain (ou un pipeline agissant en son nom) exécutait `kubectl apply -f k8s/`, et le cluster changeait d'état à cet instant précis. Rien ne garantissait que le cluster reste conforme à ce fichier après coup — une modification directe (`kubectl edit`, `kubectl scale`) pouvait faire dévier le cluster de ce que Git décrivait, sans qu'aucun outil ne le remarque.

Le GitOps inverse ce modèle en **pull** : un contrôleur (ici, ArgoCD) tourne en permanence à l'intérieur du cluster, observe un dépôt Git, et compare en continu ce que Git décrit (l'**état désiré**) à ce qui existe réellement dans le cluster (l'**état réel**). Dès qu'un écart apparaît, ArgoCD le signale — et peut le corriger automatiquement si on l'y autorise.

## Push vs pull, concrètement

| | Modèle push (Projet 4-5) | Modèle pull / GitOps (Projet 6) |
|---|---|---|
| Qui déclenche le changement | Un humain ou un pipeline, depuis l'extérieur du cluster | Un contrôleur, depuis l'intérieur du cluster |
| Accès nécessaire | Le pipeline CI a besoin des identifiants du cluster (`kubeconfig`) | Le cluster tire lui-même depuis Git — aucun identifiant cluster à donner à la CI |
| État après un changement manuel | Invisible, personne ne le sait | Détecté et affiché comme un écart (drift) |
| Source de vérité | Ambiguë (le cluster ? le dernier `kubectl apply` ? un ticket ?) | Sans ambiguïté : c'est ce qui est commité dans Git, rien d'autre |

Le fait que le pipeline n'ait plus besoin d'accès direct au cluster est un vrai gain de sécurité, pas juste un détail d'architecture : moins de systèmes détiennent des identifiants capables de modifier la production.

## État désiré vs état réel

- **État désiré** : ce que les fichiers YAML du dépôt GitOps décrivent, à un commit donné.
- **État réel** : ce qui tourne effectivement dans le cluster à un instant donné (interrogeable avec `kubectl get`).

ArgoCD ne fait qu'une chose, en boucle : comparer les deux. Un état `Synced` signifie qu'ils sont identiques. Un état `OutOfSync` signifie qu'un écart existe — soit parce que Git a changé et que le cluster n'a pas encore été synchronisé, soit parce que le cluster a été modifié directement (un **drift**).

## Pourquoi séparer le dépôt applicatif du dépôt GitOps

Deux raisons concrètes, pas juste une convention :

1. **Rythmes différents** : le dépôt applicatif change à chaque commit de code (plusieurs fois par jour). Le dépôt GitOps ne devrait changer qu'à chaque décision de déploiement (beaucoup plus rare). Les mélanger rendrait l'historique du dépôt GitOps illisible.
2. **Permissions différentes** : n'importe quel développeur peut avoir le droit de committer du code applicatif. Modifier le dépôt GitOps revient à modifier ce qui tourne en production — ça mérite des règles de protection plus strictes (revues obligatoires, branches protégées), qu'on ne veut pas forcément imposer au dépôt de code.

## Pourquoi Git devient la source de vérité

Parce que Git offre, gratuitement, tout ce qu'il faut pour gouverner un système en production : un historique complet et immuable (qui a changé quoi, quand), une revue avant fusion (Pull Request), et un mécanisme de retour en arrière natif (`git revert`). Faire de Git la source de vérité, c'est appliquer ces garanties à l'infrastructure elle-même, pas seulement au code.

## Le rôle d'ArgoCD dans ce schéma

ArgoCD est le contrôleur qui fait exister ce modèle concrètement : il clone le dépôt GitOps, rend les manifestes (bruts ici, potentiellement via Helm/Kustomize), les compare à l'état du cluster via l'API Kubernetes, affiche l'écart dans son interface, et applique la synchronisation — manuellement ou automatiquement selon la politique choisie (voir `docs/argocd-application.md`).
