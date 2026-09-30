# TROUBLESHOOTING

Ce document recense les problèmes rencontrés lors de la mise en place du cluster, des manifests et de la PKI Vault, ainsi que les solutions appliquées. Il sert de retour d'expérience et de référence pour toute personne reprenant le projet.

---

## Sommaire

1. [Azure / AKS](#1-azure--aks)
2. [Helm](#2-helm)
3. [Prometheus Operator / kube-prometheus-stack](#3-prometheus-operator--kube-prometheus-stack)
4. [Persistance Azure Disk](#4-persistance-azure-disk)
5. [Grafana](#5-grafana)
6. [cert-manager](#6-cert-manager)
7. [HashiCorp Vault](#7-hashicorp-vault)
8. [cert-manager + Vault (PKI)](#8-cert-manager--vault-pki)
9. [Récapitulatif des bonus validés](#9-récapitulatif-des-bonus-validés)

---

## 1. Azure / AKS

### 1.1 `az aks get-credentials` échoue avec "resource group not found"

**Symptôme**
Un collaborateur avec les droits admin sur le cluster AKS exécute `az aks get-credentials` et obtient une erreur `resource group not found`.

**Cause**
La CLI Azure utilise la souscription **active** de l'utilisateur (sa propre souscription student) pour chercher le Resource Group. Le RG n'existe pas dans cette souscription.

**Solution**
Préciser explicitement la souscription cible :

```bash
az aks get-credentials \
  --subscription <ID_SOUSCRIPTION> \
  --resource-group <NOM_RG> \
  --name <NOM_CLUSTER>
```

> Note : le paramètre `--resource-group` n'accepte que le **nom** du RG, pas son scope complet (`/subscriptions/.../resourceGroups/...`). Le scope complet n'est utile que pour les attributions de rôles (`az role assignment create --scope ...`).

### 1.2 Récupérer le scope complet d'un Resource Group

```bash
az group show --name "<NOM_RG>" --query id --output tsv
```

---

## 2. Helm

### 2.1 Différence entre un chart "CRDs" et un chart "opérateur"

**Symptôme**
Confusion : après avoir installé `prometheus-operator-crds`, on pense avoir installé l'opérateur Prometheus.

**Cause**
Le chart `prometheus-operator-crds` n'installe **que les CRDs** (définitions d'API). L'opérateur (contrôleur qui agit sur ces CRDs) n'est pas installé.

**Solution**
Pour une stack complète, utiliser `kube-prometheus-stack` qui inclut l'opérateur + Prometheus + Alertmanager + Grafana + node-exporter + kube-state-metrics.

---

## 3. Prometheus Operator / kube-prometheus-stack

### 3.1 Les dashboards provisionnés ne sont pas persistés

**Symptôme**
Des dashboards modifiés via l'UI Grafana disparaissent après un redémarrage du pod.

**Cause**
Les dashboards fournis par le chart sont provisionnés via des **ConfigMaps** montés en read-only dans le pod. Ils ne sont **pas** stockés dans le PVC Grafana (qui contient uniquement la base SQLite des dashboards créés via l'UI). Au redémarrage, Grafana recharge les ConfigMaps et écrase les modifications.

**Solution**
- Ne jamais modifier un dashboard provisionné via l'UI.
- Créer ses propres dashboards via l'UI (ils iront dans la base SQLite persistée).
- Ou provisionner ses dashboards via ConfigMap avec le label `grafana_dashboard: "1"` (approche "as-code").

### 3.2 Grafana est passé de Deployment à StatefulSet

**Symptôme**
`kubectl get deployments -n kowabunga-monitoring` ne montre plus Grafana. Le pod s'appelle désormais `kube-prometheus-stack-grafana-0`.

**Cause**
Ce n'est **pas une erreur**. Dès qu'on active la persistance (`grafana.persistence.enabled: true`), le chart bascule Grafana de Deployment à StatefulSet, ce qui garantit une identité stable au pod (`-0`) et un PVC attaché de façon persistante.

**Solution**
Utiliser `kubectl get sts` au lieu de `kubectl get deployments` pour lister Grafana :

```bash
kubectl get statefulset -n kowabunga-monitoring
kubectl exec -it -n kowabunga-monitoring kube-prometheus-stack-grafana-0 -- <cmd>
```

### 3.3 Erreur `exec: "cat": executable file not found` dans Grafana

**Symptôme**
Impossible d'utiliser `cat`, `ls`, `sh` dans le pod Grafana.

**Cause**
L'image Grafana fournie par le chart est une image **distroless** (sans binaires usuels), pour des raisons de sécurité.

**Solution**
- Vérifier la configuration via l'UI Grafana ou via les ConfigMaps côté Kubernetes.
- Pour du debug ponctuel, utiliser `kubectl debug` ou un pod éphémère.

---

## 4. Persistance Azure Disk

### 4.1 Quelle StorageClass choisir ?

| StorageClass | Type | Cas d'usage |
|---|---|---|
| `azurefile*` | Partages SMB/NFS (fichiers) | RWX, plusieurs pods simultanés |
| `managed*` | Disques blocs (SSD) | RWO, un seul pod (Prometheus, Grafana, Alertmanager, Vault) |

**Choix du projet** : `managed-csi` (Standard SSD) pour un TP, avec `WaitForFirstConsumer` (le disque n'est créé qu'au moment où un pod en a besoin, dans la bonne zone).

### 4.2 Configurer la persistance de la stack

Voir `persistence-values.yaml`. Chaque composant StatefulSet (Prometheus, Alertmanager, Grafana) reçoit son propre PVC via `volumeClaimTemplates`.

Vérification :
```bash
kubectl get pvc -n kowabunga-monitoring
```
Les PVC doivent être en `Bound` avec la StorageClass `managed-csi`.

---

## 5. Grafana

### 5.1 Identifiants par défaut

- Utilisateur : `admin`
- Mot de passe : `prom-operator` (ou récupérable via le secret K8s)

Récupération :
```bash
kubectl get secret -n kowabunga-monitoring kube-prometheus-stack-grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d && echo
```

---

## 6. cert-manager

### 6.1 Le certificat a une durée de validité trop courte

**Symptôme**
Le certificat expire en 1 heure.

**Cause**
Le manifeste `Certificate` définit `duration: 1h` (probablement un choix de debug initial).

**Solution**
Passer à une durée de production :

```yaml
spec:
  duration: 720h        # 30 jours
  renewBefore: 168h     # Renouvellement 7 jours avant expiration
```

### 6.2 Le secret TLS n'a pas le bon nom

Le nom du secret est défini dans `values.yaml` (`tlsSecret`). Il est référencé à la fois par le `Certificate` (qui le crée) et par l'`Ingress` (qui le consomme). **Ils doivent correspondre**, sinon l'Ingress sert un certificat obsolète ou aucun.

---

## 7. HashiCorp Vault

### 7.1 `kubectl` n'est pas disponible dans le pod Vault

**Symptôme**
```
sh: kubectl: not found
```

**Cause**
L'image Vault est minimaliste et ne contient pas `kubectl`. Elle est faite pour exécuter Vault, pas pour piloter le cluster.

**Solution**
Exécuter les commandes `kubectl` depuis la **machine locale**, et injecter les fichiers dans le pod via `kubectl exec -i` :

```bash
kubectl get configmap -n kube-system extension-apiserver-authentication \
  -o jsonpath='{.data.client-ca-file}' \
  | kubectl exec -i -n vault vault-0 -- sh -c 'cat > /tmp/k8s-ca.crt'
```

### 7.2 `jq: not found`

**Symptôme**
Impossible d'extraire un champ JSON de la sortie Vault.

**Cause**
`jq` n'est pas présent dans l'image Vault.

**Solution**
Utiliser le flag natif `-field` de Vault :

```bash
vault write -field=csr pki_int/intermediate/generate/internal \
  common_name="..." > /tmp/intermediate.csr
```

### 7.3 `invalid character '-' in numeric literal` sur `vault read pki_int/ca/pem`

**Symptôme**
Erreur de parsing JSON.

**Cause**
La CLI tente de parser en JSON une réponse qui est en **PEM brut**. Les endpoints `ca/pem` retournent du contenu brut, pas du JSON.

**Solution**
Utiliser `-format=raw` :

```bash
vault read -format=raw pki_int/ca/pem | head -20
```

Ou préférer l'endpoint `ca_chain` qui, lui, retourne du JSON :

```bash
vault read pki_int/ca_chain
```

### 7.4 Warning "no AIA fields configured" sur le mount intermédiaire

**Symptôme**
```
WARNING! The following warnings were returned from Vault:
  * This mount hasn't configured any authority information access (AIA) fields ...
```

**Cause**
Chaque **mount PKI** a sa propre configuration d'URLs AIA. On l'a configurée pour la racine (`pki`), mais pas pour l'intermédiaire (`pki_int`).

**Solution**
Configurer les URLs pour l'intermédiaire :

```bash
vault write pki_int/config/urls \
  issuing_certificates="http://vault.vault.svc:8200/v1/pki_int/ca" \
  crl_distribution_points="http://vault.vault.svc:8200/v1/pki_int/crl"
```

Non bloquant pour le TP, mais recommandé pour une PKI propre.

### 7.5 La variable `VAULT_ADDR` n'est pas définie par défaut

**Symptôme**
À chaque nouvelle session dans le pod Vault, il faut refaire `export VAULT_ADDR='http://127.0.0.1:8200'`.

**Cause**
Les variables d'environnement ne persistent pas entre sessions shell. Le chart ne les définit pas par défaut.

**Solution (optionnelle)**
Ajouter dans `vault-values.yaml` :

```yaml
server:
  extraEnvironmentVars:
    VAULT_ADDR: http://127.0.0.1:8200
```

Puis `helm upgrade`.

---

## 8. cert-manager + Vault (PKI)

### 8.1 Erreur `403 service account name not authorized` à la création du `VaultIssuer`

**Symptôme**
```
Failed to initialize Vault client: ... Code: 403. Errors:
* service account name not authorized
```

**Cause**
Incohérence entre le ServiceAccount utilisé par cert-manager (`vault-issuer`) et celui autorisé par le rôle Vault (`cert-manager`).

**Solution**
Mettre à jour le rôle Vault pour accepter le bon ServiceAccount et le bon namespace :

```bash
vault write auth/kubernetes/role/cert-manager \
  bound_service_account_names=vault-issuer \
  bound_service_account_namespaces=kowabunga-monitoring \
  policies=kowabunga-pki-policy \
  ttl=1h
```

Puis recréer le `VaultIssuer`.

### 8.2 Erreur `403 permission denied` lors du login test

**Symptôme**
```
vault write auth/kubernetes/login role=cert-manager jwt=@/tmp/test-token
→ Code: 403. Errors: * permission denied
```

**Cause**
Le `token_reviewer_jwt` configuré dans Vault n'a pas le rôle RBAC `system:auth-delegator`, nécessaire pour valider les tokens auprès de l'API Kubernetes.

Message détaillé obtenu avec `vault monitor -log-level=debug` :
```
tokenreviews.authentication.k8s.io is forbidden:
User "system:serviceaccount:cert-manager:cert-manager" cannot create resource
"tokenreviews" in API group "authentication.k8s.io" at the cluster scope:
Azure does not have opinion for this user.
```

**Cause précise**
Sur un cluster AKS avec **Azure RBAC for Kubernetes Authorization** activé, la requête passe d'abord par Azure RBAC (qui répond "No Opinion" pour une identité purement Kubernetes), puis par le RBAC Kubernetes natif (qui refuse car le ServiceAccount n'a pas la permission).

**Solution**
Créer le `ClusterRoleBinding` manquant :

```bash
kubectl create clusterrolebinding cert-manager-tokenreview \
  --clusterrole=system:auth-delegator \
  --serviceaccount=cert-manager:cert-manager
```

Puis regénérer un `token_reviewer_jwt` frais et reconfigurer Vault :

```bash
kubectl create token cert-manager -n cert-manager --duration=24h \
  | kubectl exec -i -n vault vault-0 -- sh -c 'cat > /tmp/reviewer-token'
```

```bash
vault write auth/kubernetes/config \
  kubernetes_host="https://kubernetes.default.svc" \
  kubernetes_ca_cert=@/tmp/k8s-ca.crt \
  token_reviewer_jwt=@/tmp/reviewer-token
```

### 8.3 `the common_name field is required` lors de la signature d'un certificat

**Symptôme**
```
Vault failed to sign certificate: ... Code: 400. Errors:
* the common_name field is required, or must be provided in a CSR
  with "use_csr_common_name" set to true, unless "require_cn" is set to false
```

**Cause**
Conflit entre :
- La politique par défaut de Vault (`require_cn=true`, le CN est obligatoire).
- Le comportement de cert-manager qui n'envoie **pas** de `commonName` (champ déprécié depuis 2000, remplacé par les SANs).

**Solution (recommandée)**
Désactiver l'obligation du CN dans le rôle Vault :

```bash
vault write pki_int/roles/kowabunga-role \
  allowed_domains="tortueninja.spaincentral.cloudapp.azure.com" \
  allow_subdomains=false \
  allow_bare_domains=true \
  max_ttl="720h" \
  key_type="rsa" \
  key_bits=2048 \
  require_cn=false
```

**Solution alternative**
Spécifier un `commonName` dans le manifeste `Certificate` (moins propre).

### 8.4 Le certificat ne se ré-émet pas après correction

**Symptôme**
Après avoir corrigé un problème et supprimé la `CertificateRequest` échouée, cert-manager ne crée pas de nouvelle requête.

**Cause**
cert-manager applique un **backoff exponentiel** en cas d'échec. Supprimer la `CertificateRequest` seule ne suffit pas à le relancer.

**Solution**
Invalider l'état du Certificate en supprimant le Secret TLS (ou en recréant le Certificate) :

```bash
# Méthode 1 : supprimer le secret
kubectl delete secret <nom-secret-tls> -n <namespace>

# Méthode 2 : supprimer et recréer le Certificate
kubectl delete -f <certificate.yaml>
kubectl apply -f <certificate.yaml>
```

Puis surveiller :
```bash
kubectl get certificate <nom-cert> -n <namespace> -w
```

### 8.5 Warning navigateur "Autorité non reconnue" malgré une PKI valide

**Symptôme**
Le navigateur (Firefox, Chrome) affiche un avertissement de sécurité alors que la chaîne de confiance est correcte (`Feuille → Intermediate → Root`).

**Cause**
Le certificat **n'est plus auto-signé** au sens strict : il est signé par ta `Kowabunga Intermediate CA`. Mais le navigateur remonte la chaîne jusqu'à `Kowabunga Root CA` et constate qu'elle n'est **pas dans son trust store**.

**Solution (pour faire disparaître le warning)**
Importer le certificat racine dans le trust store du navigateur / de l'OS :

```bash
# Récupérer la racine
kubectl exec -n vault vault-0 -- sh -c \
  "VAULT_ADDR='http://127.0.0.1:8200' vault read -format=raw pki/ca/pem" \
  > root-ca.pem
```

Puis :
- **Firefox** : Paramètres → Vie privée et sécurité → Certificats → Afficher les certificats → Autorités → Importer → cocher "Confirmer cette autorité pour identifier des sites web".
- **Linux** : copier dans `/usr/local/share/ca-certificates/` puis `update-ca-certificates`.
- **macOS** : Trousseau d'accès → importer + approuver.
- **Windows** : `certmgr.msc` → Autorités de certification racines de confiance → importer.

Une fois importée, `curl` sans `-k` doit fonctionner.

---

## 9. Récapitulatif des bonus validés

| Bonus | Statut | Détail |
|---|---|---|
| Persistance Azure Disk (stack complète) | ✅ | Prometheus 50Gi, Alertmanager 5Gi, Grafana 5Gi, Vault 10Gi — StorageClass `managed-csi` |
| PKI dans HashiCorp Vault | ✅ | Vault déployé en mode standalone persistant, moteur PKI racine + intermédiaire, auth Kubernetes |
| Certificat racine | ✅ | `Kowabunga Root CA` généré dans Vault (mount `pki`), TTL 10 ans |
| Certificat intermédiaire | ✅ | `Kowabunga Intermediate CA` signé par la racine, mount `pki_int`, TTL 5 ans |
| IP publique fixe pour le Load Balancer | ⚠️ | Non traité |

### Chaîne de confiance finale

```
Ingress (tortueninja.spaincentral.cloudapp.azure.com)
   │
   ▼
Certificat feuille (émis par cert-manager, signé par Vault)
   │
   ▼
Kowabunga Intermediate CA (pki_int, TTL 5 ans)
   │
   ▼
Kowabunga Root CA (pki, TTL 10 ans)
```

L'authentification de cert-manager auprès de Vault se fait via la **méthode Kubernetes** (ServiceAccount token), sans aucun secret statique stocké. La politique Vault `kowabunga-pki-policy` applique le principe du moindre privilège : seule la signature pour le domaine autorisé est permise.