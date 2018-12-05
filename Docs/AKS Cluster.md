# AKS Full Install With Docker Pull and Ingress

ToDo:

 * AKS Provisioning
 * Helm Install
 * Permission to ACR
 * Ingress
 * AKS Autoscaling
 * DevSpaces
 * Container Monitoring


#### Create ResourceGroup and start cluster

[https://docs.microsoft.com/en-us/azure/aks/kubernetes-walkthrough]()

```
RESGRP=demo-rg
CLUSTER=demo-aks
LOCATION=westeurope
REGISTRY=testrigregistry
REGISTRY_RESGRP=utils-rg

az group create --name ${RESGRP} --location ${LOCATION}
az aks create --resource-group ${RESGRP} --name ${CLUSTER} --node-count 1 --generate-ssh-keys --disable-rbac --kubernetes-version 1.11.5 --location westeurope --node-count 1 --ssh-key-value ~\.ssh\id_rsa.pub
```


#### Get Credentials

```
az aks get-credentials --resource-group ${RESGRP} --name ${CLUSTER}

```

(Set context here with kcs)


#### Test connection to cluster
```
kubectl get nodes
```

#### Helm install 
```
helm init
```

#### Create permission to pull from ACR
[https://docs.microsoft.com/en-us/azure/container-registry/container-registry-auth-aks]()

```
# Get the id of the service principal configured for AKS
CLIENT_ID=$(az aks show --resource-group $RESGRP --name $CLUSTER --query "servicePrincipalProfile.clientId" --output tsv)

# Get the ACR registry resource id
ACR_ID=$(az acr show --name $REGISTRY --resource-group $REGISTRY_RESGRP --query "id" --output tsv)

# Create role assignment
az role assignment create --assignee $CLIENT_ID --role Reader --scope $ACR_ID

```


#### Application gateway Install



#### Traefik Install

Trafik user: admin
Traefik PW: YTdtHSuWv9DJ33KWuzqFaFN39JQL5U8w











"================================================"

#### HTTP Application routing
[https://docs.microsoft.com/en-us/azure/aks/http-application-routing]()

[https://docs.microsoft.com/en-us/azure/aks/ingress-tls]()


Create NGINX load balancer

```
helm repo update
helm install stable/nginx-ingress --namespace kube-system --set controller.replicaCount=2 --set rbac.create=false
```

Get IP address and update DNS (wait for a few minutes !!)

```
# Wait until ready
watch kubectl get service -l app=nginx-ingress --namespace kube-system 

# Public IP address of your ingress controller
IP=$(kubectl get service -l app=nginx-ingress --namespace kube-system -o=json | jq -r '.| .items[0].status.loadBalancer.ingress | .[] | .ip')

# Get the resource-id of the public ip
PUBLICIPID=$(az network public-ip list --query "[?ipAddress!=null]|[?contains(ipAddress, '$IP')].[id]" --output tsv)

# Update public ip address with DNS name
az network public-ip update --ids $PUBLICIPID --dns-name $CLUSTER
```

#### Install cert-manager


Create cert-manager controller

```
helm install stable/cert-manager \
    --namespace kube-system \
    --set ingressShim.defaultIssuerName=letsencrypt-staging \
    --set ingressShim.defaultIssuerKind=ClusterIssuer \
    --set rbac.create=false \
    --set serviceAccount.create=false
```

Create a CA cluster issuer

* Create file `cluster-issuer.yaml`

```
apiVersion: certmanager.k8s.io/v1alpha1
kind: ClusterIssuer
metadata:
  name: letsencrypt-staging
spec:
  acme:
    server: https://acme-staging-v02.api.letsencrypt.org/directory
    email: jim.leitch@sunningdale.nl
    privateKeySecretRef:
      name: letsencrypt-staging
    http01: {}
```

Create the issuer from file

```
kubectl apply -f cluster-issuer.yaml
```
Create certificate object, first create file, updating `dnsNames` and `domains` with value `demo-aks.westeurope.cloudapp.azure.com` or value of ${CLUSTER}.${LOCATION}.cloudapp.azure.com

#### Chck health and container logs
[https://aka.ms/coingfonboarding]()


NOTES:

```
helm install --name self-blue --namespace self-blue helm/spr-monolith --set image.tag=master
```






















