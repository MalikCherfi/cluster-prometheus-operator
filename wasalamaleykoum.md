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