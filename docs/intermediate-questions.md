# Questions intermédiaires — Projet 6

## Jour 1 — Concepts GitOps et installation d'ArgoCD

### 11. Qu'est-ce que le GitOps ?

Une méthode où Git décrit l'état désiré d'un système (ici, un cluster Kubernetes), et où un contrôleur logiciel (ArgoCD) applique et maintient en permanence le système dans cet état, en comparant sans arrêt ce qui est décrit à ce qui existe réellement. Le déploiement n'est plus un acte ponctuel déclenché de l'extérieur (`kubectl apply`), c'est une convergence continue pilotée depuis l'intérieur du cluster.

### 12. Pourquoi séparer le dépôt applicatif du dépôt GitOps ?

Deux raisons concrètes : des rythmes de changement très différents (le code change à chaque commit, les manifestes de déploiement beaucoup plus rarement — les mélanger rend l'historique de déploiement illisible), et des besoins de protection différents (modifier ce qui tourne en production mérite des règles plus strictes que modifier du code applicatif). Détail complet : `docs/gitops-principles.md`.

### 13. Quelle est la différence entre l'état désiré et l'état réel ?

L'état désiré est ce que décrivent les fichiers YAML du dépôt GitOps à un commit donné — une intention, figée dans Git. L'état réel est ce qui tourne effectivement dans le cluster à l'instant présent — vérifiable avec `kubectl get`. Le travail entier d'ArgoCD consiste à comparer ces deux états et à signaler (ou corriger) tout écart entre eux.

### 14. Quel est le rôle d'ArgoCD ?

Cloner le dépôt GitOps, en interpréter les manifestes, les comparer à l'état réel du cluster via l'API Kubernetes, afficher clairement tout écart (`Synced`/`OutOfSync`), et appliquer la synchronisation — manuellement ou automatiquement selon la politique configurée. En interne, ce rôle est réparti entre plusieurs composants spécialisés (voir `docs/argocd-installation.md`) : le `repo-server` ne connaît que Git, l'`application-controller` ne connaît que la comparaison désiré/réel, le `server` n'expose que l'interface et l'API.

### 15. Pourquoi Git devient-il la source de vérité ?

Parce que Git fournit déjà, sans rien construire de plus, tout ce qu'il faut pour gouverner un système en production de façon fiable : un historique complet et immuable de qui a changé quoi et quand, une revue avant fusion (Pull Request), et un mécanisme de retour en arrière natif (`git revert`). Faire de Git la source de vérité pour Kubernetes, c'est étendre ces garanties à l'infrastructure elle-même, au lieu de les réserver au seul code applicatif.
