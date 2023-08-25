# Create NKE Cluster
## MetalLB

# Configure CAPX
https://opendocs.nutanix.com/capx/v1.2.x/getting_started/

Please use clusterctl v1.5+
```
clusterctl init -i nutanix
```

Add CAAPH Addon
```
clusterctl init --addon helm
```

# Vault

```
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install -f scripts/vault-values.yaml vault hashicorp/vault -n vault --create-namespace
```

Unseal the store:

```
kubectl exec -n vault vault-0 -- vault operator init \
    -key-shares=1 \
    -key-threshold=1 \
    -format=json > ./cluster-keys.json

VAULT_UNSEAL_KEY=$(jq -r ".unseal_keys_b64[]" ./cluster-keys.json)
kubectl exec -n vault vault-0 -- vault operator unseal $VAULT_UNSEAL_KEY
```

Export Root Token and Login

```
export CLUSTER_ROOT_TOKEN=$(cat ./cluster-keys.json | jq -r ".root_token")
kubectl exec -n vault vault-0 -- vault login $CLUSTER_ROOT_TOKEN
```

Interactive Vault Shell

```
kubectl exec -n vault -it vault-0 -- /bin/sh
``` 

## Prepare Vault for ESO

```
vault secrets enable --path=secret kv-v2

vault policy write demo-policy -<<EOF     
path "secret/data/*" {
  capabilities = ["create", "update", "read"]
}
path "secret/metadata/*"
{
  capabilities = [ "list", "read" ]
}
EOF

vault kv put secret/prismcentral/crimson-pc endpoint=crimson-pc.dachlab.net user=admin pass=ChangeMe

vault token create --policy=demo-policy
```

# External Secrets Operator (ESO)

```
helm repo add external-secrets https://charts.external-secrets.io

helm install external-secrets \
   external-secrets/external-secrets \
  -n external-secrets \
  --create-namespace \
  --set installCRDs=true
```

Enable vault backend
```
kubectl apply -f scripts/clustersecretstore.yaml
```

Add previously created token
```
kubectl create secret generic vault-token -n external-secrets --from-literal=token=MY_TOKEN
```

# ArgoCD
## Install

```
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd -n argocd -f scripts/argo-values.yaml

```

Edit Configmap for using Cilium Helm Chart:
```
kubectl edit configmap argocd-cm -n argocd
```

```
data:
  resource.exclusions: |
    - apiGroups:
        - cilium.io
      kinds:
        - CiliumIdentity
      clusters:
        - "*"
```

## Argocd Vault Plugin

create Secret (replace Placeholder with your Token)
```
kubectl create -f scripts/argocd-vault-plugin-credentials.yaml
```

Adding Plugin Configmap
```
kubectl apply -f scripts/argocd-vault-plugin-cmp.yaml
```

## RBAC
Argo needs privileges to create CAPI Clusters

```
kubectl create clusterrolebinding argocd-application-controller-cluster-admin-binding --clusterrole=cluster-admin --serviceaccount=argocd:argocd-application-controller
```

Create Token for API-Access (needed later on Capi2Argo Operator)
```
argocd account generate-token --account secretoperator
```

## GUI/CLI Access
Initial secret:
```
kubectl get secret -n argocd argocd-initial-admin-secret  --template={{.data.password}} | base64 -D
```

Ingress:
```
kubectl apply -f scripts/argocd-ingress.yaml
```

# Kyverno
## Install
```
kubectl create -f https://github.com/kyverno/kyverno/releases/download/v1.10.1-rc.1/install.yaml
```
## Policy
```
kubectl apply -f scripts/kyverno-policy.yaml
```

# Crossplane
Crossplane is used for additional Terraform-based Infrastructure components like Flow Polices

```
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update
helm install crossplane --namespace crossplane-system --create-namespace crossplane-stable/crossplane
```

Install Terraform Provider for Crossplane
```
kubectl apply -f scripts/crossplane.yaml
```

# Dashboard

helm repo add kubernetes-dashboard https://kubernetes.github.io/dashboard
helm repo update
helm install kubernetes-dashboard kubernetes-dashboard/kubernetes-dashboard --create-namespace --namespace kubernetes-dashboard -f scripts/dashboard-values.yaml

kubectl get secret admin-user -n kubernetes-dashboard -o jsonpath={".data.token"} | base64 -d


# Grafana

helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm install grafana grafana/grafana --create-namespace --namespace grafana-system -f scripts/grafana-values.yaml
kubectl get secret --namespace grafana-system grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo