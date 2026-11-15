# 02-calico — Romaissa

## Objectif
Déployer frontend / backend / db dans un namespace dédié, prouver qu'au départ
tout communique avec tout, puis restreindre avec Calico (NetworkPolicy) pour
n'autoriser que `frontend → backend → db` (+ DNS).

> Les 3 "applications" sont volontairement de simples `nginx:alpine` : ce qui
> compte ici, ce sont les flux réseau, pas le contenu applicatif.

## Contexte du cluster
- Cluster microk8s avec Calico déjà actif (CNI).
- Namespace de travail : **`romy`** (déjà existant, créé par l'administrateur
  du cluster — RBAC en place avec droits limités à ce namespace).
- Un **ResourceQuota** est actif sur `romy` : chaque conteneur doit déclarer
  `requests`/`limits` CPU et mémoire, sinon le pod est refusé à la création.

## Fichiers du dossier

| Fichier | Rôle |
|---|---|
| `01-frontend.yaml` | Deployment + Service frontend |
| `02-backend.yaml`  | Deployment + Service backend |
| `03-db.yaml`       | Deployment + Service db |
| `04-networkpolicy-default-deny.yaml` | Bloque tout le trafic entrant/sortant |
| `05-networkpolicy-allow-dns.yaml`    | Autorise la résolution DNS vers kube-system |
| `06-networkpolicy-frontend-to-backend.yaml` | Autorise frontend → backend (port 80) |
| `07-networkpolicy-backend-to-db.yaml`       | Autorise backend → db (port 80) |

## Déploiement
```bash
kubectl apply -f 01-frontend.yaml
kubectl apply -f 02-backend.yaml
kubectl apply -f 03-db.yaml

kubectl get pods -n romy
kubectl get svc -n romy
```

## Étape 1 — Preuve que tout communique avec tout (avant toute policy)
```bash
kubectl exec -n romy deploy/frontend -- wget -qO- --timeout=2 http://backend-svc && echo OK-backend
kubectl exec -n romy deploy/frontend -- wget -qO- --timeout=2 http://db-svc && echo OK-db
kubectl exec -n romy deploy/backend -- wget -qO- --timeout=2 http://frontend-svc && echo OK-frontend
kubectl exec -n romy deploy/backend -- wget -qO- --timeout=2 http://db-svc && echo OK-db
kubectl exec -n romy deploy/db -- wget -qO- --timeout=2 http://frontend-svc && echo OK-frontend
kubectl exec -n romy deploy/db -- wget -qO- --timeout=2 http://backend-svc && echo OK-backend
```
**Résultat obtenu :** les 6 commandes retournent la page nginx par défaut
suivie de `OK-*` → tout communique avec tout, confirmé.

## Étape 2 — Default-deny + DNS
```bash
kubectl apply -f 04-networkpolicy-default-deny.yaml
kubectl apply -f 05-networkpolicy-allow-dns.yaml
kubectl get networkpolicy -n romy
```
Relancer les mêmes 6 commandes de test.

**Résultat obtenu :** les 6 commandes retournent `wget: download timed out`
→ le default-deny bloque bien tout le trafic entre pods.

## Étape 3 — Autoriser uniquement frontend → backend → db
```bash
kubectl apply -f 06-networkpolicy-frontend-to-backend.yaml
kubectl apply -f 07-networkpolicy-backend-to-db.yaml
kubectl get networkpolicy -n romy
```
Relancer les mêmes 6 commandes de test.

**Résultat obtenu :**

| Flux | Résultat |
|---|:---:|
| frontend → backend | ✅ OK |
| frontend → db | ❌ bloqué |
| backend → frontend | ❌ bloqué |
| backend → db | ✅ OK |
| db → frontend | ❌ bloqué |
| db → backend | ❌ bloqué |

Conforme à l'objectif : seul le chemin `frontend → backend → db` passe.

## Incident rencontré et résolu : panne du plugin CNI Calico

**Symptôme :** après application des Deployments, les 3 pods restaient
bloqués indéfiniment en `ContainerCreating` (plus de 15 minutes).

**Diagnostic :**
```bash
kubectl describe pod frontend-xxxx -n romy
```
a révélé l'erreur :
```
Failed to create pod sandbox: rpc error: code = Unknown desc = failed to
setup network for sandbox "...": plugin type="calico" failed (add):
error getting ClusterInformation: connection is unauthorized: Unauthorized
```

**Cause :** le fichier `/etc/cni/net.d/calico-kubeconfig`, qui contient le
jeton d'authentification utilisé par le plugin CNI Calico pour contacter
l'API Kubernetes à chaque création de pod, avait expiré (dernière
modification datant de plusieurs jours alors que le pod `calico-node`
tournait depuis `Running` sans redémarrage).

**Vérification :** les pods `calico-node` et `calico-kube-controllers`
étaient bien à l'état `Running` dans `kube-system` — ce n'était donc ni un
crash, ni un problème RBAC classique (le `ClusterRoleBinding calico-node`
était intact), mais un jeton local expiré jamais rafraîchi automatiquement.

**Correction (avec accès `sudo` sur la VM) :**
```bash
sudo microk8s kubectl delete pod -n kube-system calico-node-xxxx
```
Le DaemonSet a recréé automatiquement le pod, ce qui a régénéré un jeton
frais dans `/etc/cni/net.d/calico-kubeconfig`. Les 3 pods applicatifs sont
immédiatement passés à `Running`.

## Recommandations

1. **Surveiller / automatiser le renouvellement du jeton CNI de Calico.**
   Le jeton local utilisé par le plugin CNI (`calico-kubeconfig`) n'est pas
   rafraîchi automatiquement par microk8s et peut expirer silencieusement,
   bloquant toute création de pod sur le node sans que `calico-node`
   lui-même ne signale d'erreur (il reste `Running`). Mettre en place une
   vérification périodique (health-check ou alerte) sur l'âge de ce fichier,
   ou un redémarrage planifié du DaemonSet `calico-node`, éviterait ce type
   de panne silencieuse en production.

2. **Étendre le principe de moindre privilège réseau à tout le cluster.**
   Les `NetworkPolicy` mises en place ici ne couvrent que le namespace
   `romy`. Pour une sécurité cohérente sur l'ensemble du cluster, il serait
   préférable d'utiliser des `GlobalNetworkPolicy` Calico (ou un modèle de
   "tiers") imposant un default-deny par défaut à la création de tout
   nouveau namespace, afin qu'aucun namespace ne reste "ouvert" par oubli de
   configuration.
