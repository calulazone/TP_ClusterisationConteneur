### Context

dev-user :
```bash
kubectl config set-context dev-user@app-cluster `
--cluster=minikube `
--user=dev-user `
--namespace=app-dev
```

qa-user :
```bash
kubectl config set-context qa-user@app-cluster `
--cluster=minikube `
--user=qa-user `
--namespace=app-staging
```

---

## Installation et Déploiement

### Prérequis
- Kubernetes cluster (minikube, EKS, GKE, etc.)
- kubectl configuré
- Kustomize (intégré dans kubectl >= 1.21)

### Commandes d'installation

1. **Appliquer les namespaces et RBAC**
```bash
kubectl apply -f app-dev.yaml
kubectl apply -f app-staging.yaml
kubectl apply -f app-prod.yaml
```

2. **Créer les utilisateurs et contexts**
```bash
# Dev user
kubectl config set-context dev-user@app-cluster --cluster=minikube --user=dev-user --namespace=app-dev

# QA user
kubectl config set-context qa-user@app-cluster --cluster=minikube --user=qa-user --namespace=app-staging
```

3. **Déployer les applications (Kustomize)**
```bash
# Dev
kubectl apply -k k8s/overlay/dev

# Staging
kubectl apply -k k8s/overlay/staging

# Prod
kubectl apply -k k8s/overlay/prod
```

4. **Vérifier le déploiement**
```bash
kubectl get namespaces
kubectl get deployments --all-namespaces | grep demo-app
kubectl get resourcequota -A
```

---

## Commandes de Test (Preuves)

### Test 1 : Vérifier la séparation des environnements
```bash
# Vérifier les namespaces
kubectl get ns | grep app-

# Vérifier les déploiements dans chaque env
kubectl get deployment demo-app -n app-dev
kubectl get deployment demo-app -n app-staging
kubectl get deployment demo-app -n app-prod
```

**Résultat attendu** : 3 déploiements identiques avec replicas différentes (dev:1, staging:2, prod:3)

### Test 2 : Vérifier les ConfigMaps et environnements
```bash
# Dev
kubectl exec -n app-dev $(kubectl get pod -n app-dev -l app=demo-app -o jsonpath='{.items[0].metadata.name}') -- printenv | grep -E 'ENV_NAME|LOG_LEVEL'

# Staging
kubectl exec -n app-staging $(kubectl get pod -n app-staging -l app=demo-app -o jsonpath='{.items[0].metadata.name}') -- printenv | grep -E 'ENV_NAME|LOG_LEVEL'

# Prod
kubectl exec -n app-prod $(kubectl get pod -n app-prod -l app=demo-app -o jsonpath='{.items[0].metadata.name}') -- printenv | grep -E 'ENV_NAME|LOG_LEVEL'
```

**Résultat attendu** :
- Dev : `ENV_NAME=dev`, `LOG_LEVEL=debug`
- Staging : `ENV_NAME=staging`, `LOG_LEVEL=info`
- Prod : `ENV_NAME=prod`, `LOG_LEVEL=warn`

### Test 3 : Vérifier les quotas de ressources
```bash
# Voir les quotas en dev
kubectl get resourcequota -n app-dev

# Voir l'utilisation
kubectl describe quota demo-app-quota -n app-dev

# Tester le dépassement (devrait échouer)
kubectl scale deployment demo-app -n app-dev --replicas=10
# Attendez une erreur ResourceQuota exceeded
```

**Résultat attendu** : Quota affiché, scaling rejeté si dépassement

### Test 4 : Vérifier les LimitRange
```bash
kubectl get limitrange -n app-dev
kubectl describe limitrange demo-app-limits -n app-dev
```

**Résultat attendu** : Limites affichées avec defaults et validation

### Test 5 : Vérifier le RBAC
```bash
# Vérifier les Roles
kubectl get roles -n app-dev
kubectl get rolebindings -n app-dev

# Tester accès dev-user sur dev (OK)
kubectl auth can-i get deployments --as=dev-user -n app-dev

# Tester accès dev-user sur staging (KO)
kubectl auth can-i get deployments --as=dev-user -n app-staging
```

**Résultat attendu** : 
- dev-user peut accéder à app-dev (yes)
- dev-user ne peut pas accéder à app-staging (no)

### Test 6 : Vérifier Pod Security Standards
```bash
# Vérifier les labels PSS sur les namespaces
kubectl get ns -L pod-security.kubernetes.io/enforce

# Essayer de déployer un pod non conforme en prod (devrait échouer)
kubectl run --image=nginx test-nonsecure -n app-prod
```

**Résultat attendu** : Pod rejeté en prod (restricted), accepté en dev (baseline)

### Test 7 : Tester l'application via HTTP
```bash
# Port-forward dev
kubectl port-forward -n app-dev svc/demo-app 8080:80 &

# Appel HTTP
curl -s http://localhost:8080 | grep -o 'ENV_NAME: [^<]*'

# Arrêter le port-forward
kill %1
```

**Résultat attendu** : `ENV_NAME: dev` affiché en HTML

---

## Tableau RBAC - Qui a le droit de faire quoi ?

| Utilisateur | Namespace | Permissions | Permissions refusées |
|---|---|---|---|
| **dev-user** | app-dev | • Créer/modifier deployments | • Accès app-staging |
| | | • Créer/modifier services | • Accès app-prod |
| | | • Voir logs | • Supprimer deployments |
| | | • Créer ConfigMaps | • Modifier RBAC |
| **qa-user** | app-staging | • Voir deployments | • Accès app-dev |
| | | • Voir pods/logs | • Accès app-prod |
| | | • Créer ConfigMaps | • Créer deployments |
| | | | • Supprimer ressources |
| **admin** | app-prod | • Toutes les permissions | • Aucune restriction |
| | (cluster-admin) | • Accès tous namespaces | |
| | | • Gestion RBAC | |

### Détails des Rôles

**developer-role (dev)**
- `verbs`: [get, list, watch, create, update, patch]
- `resources`: [deployments, services, configmaps, pods, pods/logs]
- Limité à namespace `app-dev`

**developer-role (staging)**
- `verbs`: [get, list, watch]
- `resources`: [deployments, services, configmaps, pods, pods/logs]
- Limité à namespace `app-staging`

**admin (prod)**
- Utilise `cluster-admin` (tous les droits)
- Accès à tous les namespaces

---



### ✅ Séparation des environnements
- [x] Namespaces dédiés : `app-dev`, `app-staging`, `app-prod`
- [x] Labels cohérents : `environment`, `app`, `owner`
- [x] Isolation réseau entre environnements

### ✅ RBAC (Role-Based Access Control)
- [x] Rôles par environnement : `developer-role-dev`, `developer-role-staging`, `developer-role-prod`
- [x] Utilisateurs dédiés : dev-user (dev), qa-user (staging), admin (prod)
- [x] Principe du moindre privilège

### ✅ Gestion des secrets
- [x] Utilisation de ConfigMaps pour données non sensibles
- [x] Secrets Kubernetes pour données sensibles (mots de passe, tokens)
- [x] Rotation automatique et limites d'accès

### ✅ Quotas et limites de ressources
- [x] ResourceQuota par namespace (CPU, RAM, pods)
- [x] LimitRange pour defaults requests/limits
- [x] Validation obligatoire en prod (min/max)

### ✅ Pod Security Standards (PSS)
- [x] PSS activé : baseline (dev/staging), restricted (prod)
- [x] Pods compliant : runAsNonRoot, no privilege escalation, drop capabilities
- [x] Audit et enforcement automatique

### ✅ Network Policies
- [x] Default deny en prod
- [x] Autorisations explicites : ingress depuis ingress-controller, egress DNS
- [x] Isolation inter-environnements

### ✅ Stratégie de release et rollback
- [x] RollingUpdate avec maxUnavailable=0 (zéro downtime)
- [x] Health checks (readiness + liveness probes)
- [x] Rollback automatique en cas d'échec

### ✅ Politique d'images
- [x] Registry privé sécurisé
- [x] Scan de sécurité automatique des images
- [x] Signature d'images pour intégrité
- [x] Mises à jour régulières et patchs de sécurité