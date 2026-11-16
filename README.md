# Projet Kubernetes – Sécurisation d'un cluster

Cluster microk8s 1.35 sur une VM Ubuntu 26.04, accessible via Tailscale (`ssh <prenom>@100.94.209.116`).
Chacun travaille dans son namespace avec `kubectl` (pas de sudo, pas de `microk8s kubectl`).

| Dossier | Partie | Responsable |
| --- | --- | --- |
| `01-deploiement/` | Déploiement du cluster et accès | Karlson |
| `02-calico/` | Politiques réseau Calico | Romy |
| `03-rbac/` | RBAC et identités | Selim |
| `04-pod-security/` | Sécurité des pods (Pod Security Admission) | Abou |
| `05-kube-bench/` | Audit kube-bench et corrections | Karlson |
| `06-demo-production/` | Démo finale commune | Tous |
| `rapport/` | Rapport de chaque membre | Tous |

Règles : chacun reste dans son namespace ; les YAML et les captures vont dans son dossier ; toute modification du cluster passe par Karlson (snapshot avant).
