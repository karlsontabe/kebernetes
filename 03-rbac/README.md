# 03 - RBAC et Gestion des Identités (Selim)

Ce dossier contient l'ensemble des manifestes et preuves pour la sécurisation des accès sur le cluster Kubernetes :

* `01-roles.yaml` : Création des ServiceAccounts (`sa-reader`, `sa-developer`), des `Role` et `RoleBinding` selon le principe du moindre privilège dans le namespace `selim`.
* `csr.yaml` : Demande de certificat client (`CertificateSigningRequest`) soumise via l'API Kubernetes pour l'utilisateur `selim-user`.
* `02-attacker.yaml` : Scénario de démonstration d'exfiltration de secret via le montage par défaut du token d'un ServiceAccount sur-privilégié.
* `02-secured.yaml` : Remédiation appliquant `automountServiceAccountToken: false` et le rôle restreint `sa-reader`.
* `captures/` : Captures d'écran des tests `kubectl auth can-i`, de l'approbation du CSR et du blocage de l'attaque (`403 Forbidden`).
