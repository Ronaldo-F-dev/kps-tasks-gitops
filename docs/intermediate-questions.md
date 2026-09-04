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

## Jour 3 — Synchronisation automatisée, changement de version et drift

### 42. Quelle est la différence entre synchronisation manuelle et automatisée ?

En mode manuel, ArgoCD détecte et affiche l'écart (`OutOfSync`) mais attend une confirmation explicite (`argocd app sync`) avant d'agir. En mode automatisé (`--sync-policy automated`), il applique le changement dès qu'il le détecte, sans confirmation. Nuance testée en vrai sur ce projet : l'automatisation seule ne concerne que les changements **venant de Git** — elle n'a pas empêché ni corrigé automatiquement le drift provoqué manuellement, parce que l'option séparée `self-heal` n'était pas activée.

### 43. Qu'est-ce qu'un drift ?

Un écart entre l'état désiré (Git) et l'état réel (cluster), causé par une modification faite **directement** dans le cluster plutôt que via Git — par exemple un `kubectl scale` tapé dans l'urgence. Techniquement, ArgoCD affiche exactement le même signal (`OutOfSync`) que pour un simple changement de version en attente ; le mot "drift" décrit la cause humaine, pas un état technique différent.

### 44. Pourquoi modifier directement le cluster est-il risqué ?

Parce que ce changement devient invisible pour tout le monde tant que personne ne va vérifier ArgoCD activement : aucune trace dans l'historique Git, aucune revue possible, et le prochain déploiement normal depuis Git pourrait silencieusement écraser ce changement (ou, à l'inverse, le changement pourrait rester en place indéfiniment sans que Git ne le sache jamais). Le comportement du système devient imprévisible dès que deux sources de vérité coexistent.

### 45. Comment ArgoCD détecte-t-il un écart ?

L'`application-controller` (voir Jour 1) compare en continu, champ par champ, le contenu rendu des manifestes Git à l'état retourné par l'API Kubernetes pour les mêmes objets. Dès qu'un champ diffère (une image, un nombre de réplicas, un label...), l'objet concerné passe `OutOfSync`, et le diff exact est consultable (`argocd app diff`).

### 46. Que se passe-t-il si Git contient une mauvaise configuration ?

ArgoCD l'applique quand même — il n'a aucun moyen de juger si une configuration est "bonne" ou "mauvaise" fonctionnellement, il ne vérifie que la cohérence syntaxique et la conformité à l'API Kubernetes. Une erreur de logique commitée dans Git (mauvais tag d'image, mauvaise valeur de configuration) sera fidèlement synchronisée, et le symptôme apparaîtra ensuite dans le `Health Status` (`Degraded`) ou dans le comportement réel de l'application. C'est pour ça que le dépôt GitOps mérite une revue avant fusion, exactement comme du code.

### 47. Pourquoi le dépôt GitOps doit-il être protégé ?

Parce qu'il est devenu, dans ce modèle, l'équivalent direct de la production : n'importe quel commit fusionné dessus finit par tourner réellement sur le cluster, automatiquement. Sans protection de branche (revue obligatoire, restriction de qui peut fusionner), le dépôt GitOps serait une porte d'entrée directe vers la production, sans le moindre contrôle humain — bien plus sensible qu'un dépôt de code applicatif classique.

## Jour 3 — Synchronisation automatisée, changement de version et drift

### 42. Quelle est la différence entre synchronisation manuelle et automatisée ?

En mode manuel, ArgoCD détecte un écart (`OutOfSync`) mais attend qu'un humain déclenche `argocd app sync`. En mode automatisé, dès qu'ArgoCD détecte que Git a changé, il synchronise seul, sans attendre. Nuance importante testée en pratique : le mode automatisé ne surveille par défaut que "Git a-t-il changé ?" — pas "le cluster a-t-il été modifié directement ?". Cette seconde surveillance est un réglage séparé, `self-heal` (question 46 et suivante).

### 43. Qu'est-ce qu'un drift ?

Un écart entre l'état désiré (Git) et l'état réel (cluster) causé par une modification faite **directement** dans le cluster, en dehors de tout commit Git. Techniquement, ArgoCD l'affiche exactement comme n'importe quel autre `OutOfSync` — le mot "drift" désigne la cause (une main humaine dans le cluster), pas un état technique différent.

### 44. Pourquoi modifier directement le cluster est-il risqué ?

Parce que ce changement devient invisible pour toute personne qui ne consulte que Git — qui est censé être la source de vérité. Le risque n'est pas seulement "ArgoCD va s'en apercevoir" : c'est que, en attendant, l'équipe entière travaille avec une vision fausse de ce qui tourne réellement en production, et qu'un prochain déploiement normal (`argocd app sync`) effacera silencieusement ce changement fait dans l'urgence, sans que personne ne comprenne pourquoi.

### 45. Comment ArgoCD détecte-t-il un écart ?

L'`application-controller` (un des composants installés au Jour 1) interroge en continu l'API Kubernetes pour connaître l'état réel de chaque ressource qu'il gère, et le compare au contenu du dépôt Git à la révision suivie. La comparaison se fait objet par objet, champ par champ — c'est pour ça qu'un seul `Deployment` peut être `OutOfSync` pendant que tout le reste de l'application reste `Synced`.

### 46. Que se passe-t-il si Git contient une mauvaise configuration ?

ArgoCD l'applique quand même — il ne juge pas si le contenu de Git est "correct", seulement s'il est appliqué fidèlement. Une mauvaise configuration committée par erreur (mauvais tag d'image, mauvais port) sera synchronisée exactement comme une bonne, et l'application passera potentiellement en `Degraded`. C'est précisément pour ça que la revue avant fusion (Pull Request) sur le dépôt GitOps compte davantage que sur un dépôt de code habituel : ArgoCD n'est pas un filet de sécurité contre une erreur humaine dans Git, il en est au contraire l'exécutant fidèle.

### 47. Pourquoi le dépôt GitOps doit-il être protégé ?

Parce qu'y committer directement revient, de fait, à déployer en production — sans les vérifications habituelles (tests, revue) qu'on exigerait pour un changement de code. Une branche protégée (revue obligatoire avant fusion sur `main`) rétablit ce garde-fou : le dépôt GitOps a beau être "juste des fichiers YAML", son contenu a un pouvoir d'action direct et immédiat sur un système réel.
