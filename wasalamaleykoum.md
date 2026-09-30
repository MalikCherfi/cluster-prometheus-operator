# Wa Aleykoum Salam !!!

## Kowabunga le cris, des ninjas !! (AKS)

déjà pour commencer après avoir créé le cluster de con là il faut activer le rbac azure dessus pour que ce soit plus simple donc on update avec cette commande :  
```bash
az aks update --resource-group Leith_letudiant --name kowabungalecridesninjas --enable-aad --aad-admin-group-object-ids d4285cc9-08aa-4dea-9c63-c009efae40b5 --aad-tenant-id a2e466aa-4f86-4545-b5b8-97da7c8febf3 --disable-local-accounts --enable-azure-rbac
```  
et il faut mettre malik dans le groupe admin (de l'id correspondant à celui de la commande)  
et après le classico `az aks get-credentials` avec rg et nom aks (et sur wsl supplement ` export KUBECONFIG=/mnt/c/Users/Utilisateur/.kube/config`)

ah ouais et btw, pour update le nombre de pools (pcq j'ai crée le cluster avec un seul) c'est pas az aks update, la commande c'est `az aks nodepool scale   --resource-group Leith_letudiant   --cluster-name kowabungalecridesninjas   --name nodepool1   --node-count 2` en gros

## quatre tortues d'enfer, dans la ville !! (CRD)

donc maintenant je installe prometheus operateur sur le cluster apparament, ET AVEC HELM EN PLUS !!  
donc une fois que helm et installé, attention ça va aller vite:  
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```  
ensuite on peut check les dernieres versions comme ça : `helm search repo prometheus-operator-crds`  
et donc après en faisant juste ça  
```bash
helm install prometheus-operator-crds prometheus-community/prometheus-operator-crds \
  --namespace kowabunga-monitoring \
  --create-namespace
```  
y a prometheus operator qui est installé sur le cluster (mais j'ai rien capté (j'avais vraiment rien capté, là y a juste les CRD installés, pas l'operator lui même))  
on peut voir les CRD (custom resources definitions) comme ça `kubectl get crds | grep monitoring.coreos.com`, même si j'ai aucune idée d'à quoi ça correspond  



## chevaliers d'écailles et, de vinyle !!! (prometheus operator)

En gros j'ai installé les CRD tous seuls, et ça c'est nul ! ça sert à rien !  
Donc on va uninstall, comme ça, ça continue le tuto helm, et après on va installer le bon package helm qui contiens tout ce qu'il faut dedans 😎  
alors uninstall : `helm uninstall prometheus-operator-crds -n kowabunga-monitoring` voilà c'est facile, et pour installer le bon package c'est les mêmes commandes qu'en haut mais faut remplacer `prometheus-operator-crds` par `kube-prometheus-stack` (bref si j'avais mieux lu la page d'install de prometheus operator, ce serait allé plus vite).  

Et purée de pomme de terre, cette stack elle a installé un milliard de choses, on peut les voir avec kubectl get deployments, et get pods (dans le bon namespace)  

et après cette verivication je me rends compte que helm à fait 4 des étapes du tp pour moi, merci helm, 5 étapes même !! wesh 6 etapes carrement !! ça fume tout !!!

## ce sont des guerriers fantastiques, Ils sortent les nunchakus, c'est la panique !!!!!! (persistence des données ? peut etre)

qui se rappelle de `kubectl get storageclass` ??? pas moi mais ça me donne les types de storage dispo pour la persistence des données par exemple  
On crée un poti yaml avec les parametres pour la persistence et on l'applique avec HELM !! encore lui fumier !  
(comme ça `helm upgrade kube-prometheus-stack prometheus-community/kube-prometheus-stack  --namespace kowabunga-monitoring -f persistence-values.yaml`)  
on peut observer les volumes qui ont été créé avec pvc (`kubectl get pvc -n kowabunga-monitoring`), et ils sont visible dans la ressource aks azure aussi sur le portail (dans ressources kubernetes > stockage).

## tortues ninja ! tourntues ninjas ! tortues ninja ! tortues ninja !

Bon là j'sais plus ce que ce fais, en gros

Si tu veux impressionner le jury, tu peux mentionner dans ton README :

    "Nous avons choisi un routage path-based car le domaine cloudapp.azure.com ne permet pas la création de sous-domaines. En production, nous aurions utilisé un domaine dédié avec des sous-domaines (grafana.kowabunga.io, prometheus.kowabunga.io) pour bénéficier d'une meilleure isolation."

Pour vérifier les travaux du stagiaire, ya les classique:  
```bash
kubectl get pods -n cert-manager
kubectl get crds | grep cert-manager
kubectl get clusterissuers -o wide
kubectl get certificates -A
kubectl get certificaterequest,order,challenge -A
kubectl get secret motdepasseducertbeaucouptroplongmaissecuredeoufavecunarobasealafinmaispasvraimentparcequonapasledroitdelemettrealafin -o wide
kubectl get secret motdepasseducertbeaucouptroplongmaissecuredeoufavecunarobasealafinmaispasvraimentparcequonapasledroitdelemettrealafin -n kowabunga-monitoring -o jsonpath='{.data.tls\.crt}' | base64 -d | openssl x509 -noout -text
kubectl get svc -n ingress-nginx ingress-nginx-controller -o wide
kubectl get ingress -A
kubectl describe ingress monitoring -n kowabunga-monitoring
```

j'sais pas à quoi ça sert mais ça verifie bien 👍


## Tortues Ninjas, Tortues Ninjas Tortues Ninjas, Tortues Ninjas Tortues Ninjas, Tortues Ninjas KOWABUNGA le cri des ninjas Quatre tortues d'enfer dans la ville Chevaliers d'écailles et de vinyle Ce sont des guerriers fantastiques Ils sortent les nunchakus c'est la panique, Tortues Ninjas, Tortues Ninjas Tortues Ninjas, Tortues Ninjas Pour venger Splinter, ils sortent les katanas Ils sont les meilleurs et font la loi Mais quand il s'agit d's'amuser Finie la terreur, on est là pour rigoler (x2) Tortues Ninjas, Tortues Ninjas Tortues Ninjas, Tortues Ninjas Tortues Ninjas, Tortues Ninjas KOWABUNGA le cri des ninjas Splinter est leur maître leur chef leur professeur C'est l'ennemi fatal du sinistre Schreider C'est grâce aux chevaliers d'écailles Qu'un jour il vaincra dans la dernière bataille (x2) Tortues Ninjas, Tortues Ninjas Tortues Ninjas, Tortues Ninjas Tortues Ninjas, Tortues Ninja KOWABUNGA le cri des ninjas Quatre tortues d'enfer dans la ville Chevaliers d'écailles et de vinyle Ce sont des guerriers fantastiques Ils sortent les nunchakus, c'est la panique Tortues Ninjas, Tortues Ninjas Tortues Ninjas, Tortues Ninjas Tortues Ninjas, Tortues Ninjas KOWABUNGA le cri des ninjas

Donc ce chapitre est dédié au bonus sur le PKI, il ya plusieurs étapes, mais ça pourrait casser le systeme de certifs qu'on a de base, donc peut être creer un autre cluster ? ou risquer de tout casser c'est pas mal.  
Monsieur Deepseek, me propose un plan en 6 étapes : Déployer Vault, Initialiser et Unseal Vault, Configurer le Moteur PKI, Configurer l'Authentification Kubernetes, Créer un VaultIssuer, et Utiliser le VaultIssuer

### Déployer Vault

ça commence par mettre le repo helm de hashicorp dans helm pcq vault c'est hashicorp  
```bash
helm repo add hashicorp https://helm.releases.hashicorp.com
helm repo update
```  
puis on crée le vault avec le fichier value pour les aprametres (et dans un namespace special qu'on a créé)  
```bash
helm install vault hashicorp/vault \
  --namespace vault \
  -f vault-values.yaml
```  
quand on get les pods on voit le pod du vault-0 est running mais pas ready, et c'est normal parce qu'il est scéllé(封印)

### Initialiser et Unseal Vault

askip faut générer des clés qui vont proteger le vault. donc on commence `kubectl exec -n vault -it vault-0 -- vault operator init`, et là boum ça m'affiche 5 clé de unseal ??? c'est quoi ça ?? (et un token inital root)  
idealement faut les garder dans un gestionaire de mot de passe ou quoi, mais vu que je tiens àc e que notre infra soit le moins sécuriser possible, je vais les écrire dans ce fichier par pur esprit de contradiction, donc les voici :  
```
Unseal Key 1: j9kK37uvwoBqv4m0Sr49F+CqUN7WPloF/QuAX+qhV+Pd
Unseal Key 2: I0XnhZVu9xfHOGJyFWdDVoHZvf/0LwIyR3GA5ZTE/dyb
Unseal Key 3: be/mOoiWqc9Py8pD7nPCI4Wym+lNqfrVKX7gendvdDW/
Unseal Key 4: mp4QOuiYPYi+veV3K1QiuqrouQN67i9D/S9dmj0cggBP
Unseal Key 5: NbT9FlMoP7rtqov8W5kLW35EK9Zr46w1GkM0sSgNdxTY

Initial Root Token: hvs.8OWWoLHVQmuCHEXJnc7DtzVS
```  

donc pour desceller je dois mettre cette commande `kubectl exec -n vault -it vault-0 -- vault operator unseal` 3 fois puis coller 3 des 5 clés de déscellement.  
Après la 3eme commande on voit que le satut sealed est false, et après on peut check que le pod est ready aussi 👍

on peut verifier avec un port forward rapido (genre `kubectl port-forward -n vault svc/vault 8200:8200`) que l'UI est là et le pod fonctionne bien (on se connecte avec le initial root token)

### Configurer le Moteur PKI

donc là c'est l'etape ou on fait en plus le CA racine auto-signée, et le CA intermediaire normalement.  
purée de pomme de terre, en fait y a genre 150 étapes c'est complicado à consigner tout ça.....  
je vais essayer de faire concis :  
déjà tout se passe dans le pod du vault, donc `kubectl exec -n vault -it vault-0 -- sh`  
ensuite on se login au vault (avec le token inital root) `export VAULT_ADDR='http://127.0.0.1:8200' && vault login`  
puis on active le moteur pki du vault hashi au path /pki qui consistera à notre CA racine `vault secrets enable -path=pki pki`  
on configure la durée de vie max de la racine `vault secrets tune -max-lease-ttl=87600h pki` (10 ans)  
et là on genere le CA racine `vault write pki/root/generate/internal common_name="Kowabunga Root CA" ttl=87600h`  
alors j'suis pas sûr de pourquoi c'est obligé mais faut conigurer des url racine (notion avancée) `vault write pki/config/urls issuing_certificates="http://vault.vault.svc:8200/v1/pki/ca" crl_distribution_points="http://vault.vault.svc:8200/v1/pki/crl"`  
J'ai pas tout suivi mais là on arrive sur la partie CA intermediaire, donc un peu commle avant on fait `vault secrets enable -path=pki_int pki` et `vault secrets tune -max-lease-ttl=43800h pki_int`  
et là j'commence à être perdu faut faire un CSR ?? dans l'intermediaire `vault write -field=csr pki_int/intermediate/generate/internal common_name="Kowabunga Intermediate CA" > /tmp/pki_intermediate.csr`  
et envoyer le CSR à la racine pour qu'elle le signe ???? `vault write -field=certificate pki/root/sign-intermediate csr=@/tmp/pki_intermediate.csr format=pem_bundle ttl=43800h > /tmp/intermediate.cert.pem` ça commence à être des commandes de dégénérés incomprehensible....  
Importer le certificat signé dans l'intermédiaire `vault write pki_int/intermediate/set-signed certificate=@/tmp/intermediate.cert.pem`  
on peut mettre les url de CRL et AIA aussi dans l'intermediaire pour que ce soit plus propre (et c'est obligé dans un cas réel, mais dans ce tp on aurait pu s'en passer) `vault write pki_int/config/urls issuing_certificates="http://vault.vault.svc:8200/v1/pki_int/ca" crl_distribution_points="http://vault.vault.svc:8200/v1/pki_int/crl"`  
ok l'etape suivante c'est "Créer un rôle de signature" azi, ok, j'accepte, si tu le dis `vault write pki_int/roles/kowabunga-role allowed_domains="tortueninja.spaincentral.cloudapp.azure.com" allow_subdomains=false allow_bare_domains=true max_ttl="720h"  key_type="rsa" key_bits=2048`  
on verifie que tout a bien fonctionné avec ça `vault write pki_int/issue/kowabunga-role common_name="tortueninja.spaincentral.cloudapp.azure.com" ttl="24h"` normalement y a plein de trucs qu'on lit  
donc là on en est là dans le pod du vault  
```
pki/                          pki_int/
├── CA racine (10 ans)   ──signe──>  ├── CA intermédiaire (5 ans)
├── URLs configurées                  ├── Chaîne complète disponible
                                      ├── URLs configurées
                                      └── Rôle "kowabunga-role"
                                          └── Peut signer pour tortueninja.spainetc
```  

### Configurer l'Authentification Kubernetes

normalement ce sera plus court, mais j'pense ça va être incomprehensible pareil :  

```bash
# 1. Écrire le CA Kubernetes dans le pod
kubectl get configmap -n kube-system extension-apiserver-authentication \
  -o jsonpath='{.data.client-ca-file}' \
  | kubectl exec -i -n vault vault-0 -- sh -c 'cat > /tmp/k8s-ca.crt'

# 2. Écrire le token du ServiceAccount dans le pod
kubectl create token cert-manager -n cert-manager --duration=1h \
  | kubectl exec -i -n vault vault-0 -- sh -c 'cat > /tmp/cert-manager-token'
```  
ensuite faut activer l'authentification avec kubernetes `vault auth enable kubernetes` (donc depuis l'interieur du pod vault encore comme avant) ça active un point d'entrée /auth/kubernetes dans Vault, ok  
Configurer la connexion à l'API Kubernetes pour que Vault puisse savoir comment joindre l'API Kubernetes pour valider les tokens `vault write auth/kubernetes/config kubernetes_host="https://kubernetes.default.svc" kubernetes_ca_cert=@/tmp/k8s-ca.crt token_reviewer_jwt=@/tmp/cert-manager-token` (j'ai rien compris toujours)  
après il faut créer la politique Vault pour cert-manager qui définit ce qu'un client authentifié a le droit de faire  
```bash
vault policy write kowabunga-pki-policy - <<EOF
path "pki_int/sign/kowabunga-role" {
  capabilities = ["create", "update"]
}

path "pki_int/issue/kowabunga-role" {
  capabilities = ["create"]
}
EOF
```  
apparement ensuite faut creer un role vault pour cert-manager `vault write auth/kubernetes/role/cert-manager bound_service_account_names=cert-manager bound_service_account_namespaces=cert-manager policies=kowabunga-pki-policy ttl=1h`  
on peut verifier que tout est ok avec `vault read auth/kubernetes/config` et `vault read auth/kubernetes/role/cert-manager` (et `vault policy read kowabunga-pki-policy`)  
et en fait il manquait une étape, pour que ça fonctionne faut donner le role "system:auth-delegator" à cert-manager `kubectl create clusterrolebinding cert-manager-tokenreview --clusterrole=system:auth-delegator --serviceaccount=cert-manager:cert-manager`  
et après dans le pod toujours on peut faire la verif `vault write auth/kubernetes/login role=cert-manager jwt=@/tmp/test-token`

on en censé en être là, mais j'ai un peu de erreurs dans ma verification là (le fix c'etait le role deux ligne plus haut)
```
auth/kubernetes/
├── config → Pointe vers l'API K8s, CA et token reviewer configurés
└── role/cert-manager
    ├── bound_service_account_names: cert-manager
    ├── bound_service_account_namespaces: cert-manager
    └── policies: [kowabunga-pki-policy]

policy/kowabunga-pki-policy
├── pki_int/sign/kowabunga-role → create, update
└── pki_int/issue/kowabunga-role → create
```

### Créer un VaultIssuer

on cree le service account vault-issuer `kubectl apply -f vault-issuer-sa.yaml`  
on cree un petit role qu'on attribue au cert-manager `kubectl apply -f vault-issuer-rbac.yaml`  
on cree le issuer (j'sais pas trop ce que c'est, ce qu'il fait) `kubectl apply -f vault-issuer.yaml`  
y a un problème : ServiceAccount qui tente de s'authentifier auprès de Vault (vault-issuer) n'est pas celui que le rôle Vault cert-manager est autorisé à accepter, donc on modifie, donc on modifie le rôle Vault pour qu'il accepte le ServiceAccount vault-issuer.  
pour ça on se reconnecte au pod du vault et on modifie comme ça  
```bash
# Dans le shell du pod Vault
vault write auth/kubernetes/role/cert-manager \
  bound_service_account_names=vault-issuer \
  bound_service_account_namespaces=kowabunga-monitoring \
  policies=kowabunga-pki-policy \
  ttl=1h
```  
on peut supprimer le vault issuer et le re appliquer pour être sûr que c'est bien passé `kubectl delete -f vault-issuer.yaml` et `kubectl apply -f vault-issuer.yaml`  
là quand on verifie, y a pas d'erreur normalement (`kubectl describe issuer vault-issuer -n kowabunga-monitoring`)  
on a un autre probleme notre vault demande un common_name pour repondre à la demande de signature (si j'ai bien compris) et cert manager n'en envoie pas (en meme temps c'est deprecié depuis l'an 2000 askip), donc on va re rentrer dans le pod vault et lui dire, c'est bon trkl pas besoin du cname  
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
maintenant si on applique notre certificat test `kubectl apply -f test-certificate.yaml`  
et qu'on le check `kubectl get certificate vault-test-cert -n kowabunga-monitoring` on voit que le ready est true  
CA VEUT DIRE QUE LE PKI FONCTIONNE JE CROIS (source: deepseek)  

### Utiliser le VaultIssuer

ok donc si j'ai bien compris maintenant c'est le moment où je poeux tout casser, en gros modifier le certificate pour qu'il utilise le VaultIssuer au lieu du ClusterIssuer self-signed (hihi)

