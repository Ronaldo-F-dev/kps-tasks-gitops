# Installation d'ArgoCD — Jour 1

## Installation

ArgoCD a été installé dans son propre namespace, avec les manifestes officiels du projet :

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

### Incident rencontré : une CRD trop volumineuse pour `kubectl apply`

La première tentative a échoué sur la dernière ressource du manifeste :

```
The CustomResourceDefinition "applicationsets.argoproj.io" is invalid: metadata.annotations:
Too long: may not be more than 262144 bytes
```

**Cause** : `kubectl apply` (client-side) stocke la dernière configuration appliquée dans une annotation (`kubectl.kubernetes.io/last-applied-configuration`) pour calculer les diffs futurs. Les CRD d'ArgoCD sont volumineuses, et cette annotation dépasse la limite de taille imposée par l'API Kubernetes sur les annotations.

**Correctif** : appliquer en mode *server-side* (`--server-side`), qui ne dépend pas de cette annotation — le serveur API calcule lui-même le diff :

```bash
kubectl apply -n argocd --server-side --force-conflicts \
  -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

`--force-conflicts` est nécessaire ici parce qu'une première tentative (partiellement échouée) avait déjà laissé certains champs marqués comme gérés par le client précédent.

## Vérification des pods (tâche 5)

```bash
kubectl get pods -n argocd
```

Sept pods, tous `Running 1/1` après quelques minutes de démarrage (ArgoCD est nettement plus lourd que les composants k3s de base vus au Projet 5) :

| Pod | Rôle |
|---|---|
| `argocd-server` | Sert l'interface web et l'API — c'est lui qu'on contacte depuis un navigateur ou le CLI `argocd` |
| `argocd-repo-server` | Clone les dépôts Git et génère les manifestes Kubernetes à partir de leur contenu |
| `argocd-application-controller` | Le cœur du GitOps : compare en continu l'état désiré (Git) à l'état réel (cluster), calcule les diffs |
| `argocd-applicationset-controller` | Génère des Applications ArgoCD à partir de patrons (non utilisé dans ce projet, une seule Application suffit) |
| `argocd-dex-server` | Gère l'authentification SSO (non configurée ici — authentification locale `admin` utilisée) |
| `argocd-redis` | Cache utilisé en interne par les autres composants |
| `argocd-notifications-controller` | Envoie des notifications sur des événements ArgoCD (non configuré) |

Preuve complète : `evidence/argocd-pods.txt`.

## Accès à l'interface (tâche 6)

Le `Service` `argocd-server` est `ClusterIP` par défaut — non joignable depuis l'extérieur. Deux options existaient : `kubectl port-forward` (temporaire, un tunnel actif par session) ou exposer durablement en `NodePort` (comme pour l'application au Projet 5). Le second choix a été retenu, pour un accès permanent sans garder un terminal ouvert :

```bash
kubectl patch svc argocd-server -n argocd -p \
  '{"spec": {"type": "NodePort", "ports": [
      {"name":"http","port":80,"targetPort":8080,"nodePort":30880},
      {"name":"https","port":443,"targetPort":8080,"nodePort":30843}
  ]}}'
```

Interface accessible sur `https://<IP-VPS>:30843` (certificat auto-signé par défaut — avertissement navigateur normal, à accepter).

Pour un accès ponctuel sans modifier le Service, la méthode recommandée par le brief reste :
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

## Mot de passe admin initial (tâche 7) — et un vrai incident de sécurité

Le mot de passe généré à l'installation est stocké dans un Secret :
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d
```

**Incident réel** : cette commande a été exécutée dans un terminal partagé (session d'assistance), affichant le mot de passe en clair dans un historique visible par un tiers. Traité immédiatement comme un secret compromis, sans attendre de preuve d'utilisation malveillante :

1. Installation du CLI `argocd` (`~/bin/argocd`, sans droits root nécessaires)
2. Connexion avec l'ancien mot de passe, puis changement immédiat via `argocd account update-password`
3. Le nouveau mot de passe n'a été écrit que dans un fichier local au VPS (`~/.argocd-admin-password`, permissions `600`), jamais affiché dans un terminal partagé
4. Suppression du Secret `argocd-initial-admin-secret` une fois le mot de passe changé — recommandation officielle d'ArgoCD, pour qu'aucune trace de l'ancien mot de passe ne subsiste dans le cluster

**Prévention pour la suite** : ne jamais faire transiter un secret réel (même à usage interne comme un mot de passe d'outil d'administration) par un canal partagé ou journalisé — le générer, l'utiliser, le stocker directement là où il doit vivre, sans étape intermédiaire visible.

## Composants d'ArgoCD (tâche 8)

Voir le tableau ci-dessus. Ce qui structure le fonctionnement d'ArgoCD à retenir : le `repo-server` ne connaît que Git, l'`application-controller` ne connaît que la comparaison désiré/réel, le `server` n'est qu'une façade (UI + API) au-dessus de ces deux-là — trois responsabilités bien séparées, comme le control plane de Kubernetes lui-même (Projet 5).
