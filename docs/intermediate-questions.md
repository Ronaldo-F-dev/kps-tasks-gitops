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

## Jour 2 — Application ArgoCD et première synchronisation

### 26. Qu'est-ce qu'une Application ArgoCD ?

Un objet Kubernetes (une ressource personnalisée, `Application`, dans le namespace `argocd`) qui relie trois informations : un dépôt Git source, un chemin de manifestes à l'intérieur de ce dépôt, et une destination (cluster + namespace). C'est cet objet qu'ArgoCD surveille en continu pour savoir quoi comparer à quoi.

### 27. Que signifie *Synced* ?

L'état réel du cluster correspond exactement à ce que décrivent les manifestes du dépôt, à la révision Git actuellement suivie. Aucune action n'est nécessaire.

### 28. Que signifie *OutOfSync* ?

Un écart existe entre l'état désiré (Git) et l'état réel (cluster) — soit parce que Git a changé depuis la dernière synchronisation (cas normal, en attente de sync), soit parce que le cluster a été modifié directement en dehors de Git (un drift, voir Jour 3).

### 29. Que signifie *Healthy* ?

Les objets Kubernetes de l'application non seulement existent tels que décrits, mais fonctionnent correctement selon les critères propres à leur type — pour un `Deployment`, ça veut dire que le nombre de réplicas désiré est bien `Running` et prêt (probes passées). `Synced` et `Healthy` sont deux informations indépendantes : un objet peut être `Synced` (conforme à Git) tout en étant `Degraded` (cassé en pratique), par exemple si l'image référencée dans Git elle-même ne démarre pas.

### 30. Pourquoi ArgoCD doit-il connaître le dépôt, le chemin et le namespace ?

Parce que ces trois informations définissent, à elles seules, tout le périmètre de ce qu'une Application gère : *quoi* synchroniser (dépôt + chemin) et *où* le déployer (cluster + namespace). Sans elles, ArgoCD n'aurait aucun moyen de savoir quels objets Kubernetes lui appartiennent et lesquels appartiennent à autre chose.

### 31. Quelle est la différence entre créer une ressource avec ArgoCD et avec `kubectl apply` ?

Le résultat immédiat est identique (les mêmes objets Kubernetes créés), mais pas le mécanisme derrière : `kubectl apply` est un acte ponctuel, exécuté par un humain ou un pipeline, qui ne laisse aucune garantie que l'état créé perdure. Une ressource créée par ArgoCD reste sous sa surveillance continue — si elle est modifiée ou supprimée ensuite en dehors de Git, ArgoCD le détecte (`OutOfSync`) au lieu de l'ignorer silencieusement.
