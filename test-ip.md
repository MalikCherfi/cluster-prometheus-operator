export MC_RG="MC_Leith_letudiant_kowabungalecridesninjas_spaincentral"

# IP Azure statique (doit rester identique tout du long)
az network public-ip show -g $MC_RG -n pip-tortueninja --query ipAddress -o tsv

# IP vue côté Kubernetes
IP_AVANT=$(kubectl get svc ingress-nginx-controller -n ingress-nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "IP avant suppression : $IP_AVANT"

---

kubectl delete svc ingress-nginx-controller -n ingress-nginx

--- 


kubectl get svc -n ingress-nginx     # ingress-nginx-controller ne doit plus apparaître

az network public-ip show -g $MC_RG -n pip-tortueninja \
  --query "{ip:ipAddress, allocation:publicIPAllocationMethod}" -o table


--- 

helm upgrade ingress-nginx ingress-nginx/ingress-nginx -n ingress-nginx \
  --reuse-values -f ./helm/ingress-nginx-values.yaml

---

kubectl get svc ingress-nginx-controller -n ingress-nginx -w

---

IP_APRES=$(kubectl get svc ingress-nginx-controller -n ingress-nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "IP avant  : $IP_AVANT"
echo "IP après  : $IP_APRES"

if [ "$IP_AVANT" = "$IP_APRES" ]; then
  echo "✅ IP identique : la suppression/recréation n'a pas fait perdre l'IP fixe"
else
  echo "❌ IP différente : problème dans la config loadBalancerIP/annotations"
fi


---

curl -vk --resolve tortueninja.spaincentral.cloudapp.azure.com:443:$IP_APRES \
  https://tortueninja.spaincentral.cloudapp.azure.com/ 2>&1 | grep -E "subject|issuer|HTTP/"