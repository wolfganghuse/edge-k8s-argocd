# Create NKE Cluster
## MetalLB

# Configure CAPX
https://opendocs.nutanix.com/capx/v1.2.x/getting_started/

```
clusterctl init -i nutanix
```

## CAAPH
```
kubectl apply -f https://github.com/kubernetes-sigs/cluster-api-addon-provider-helm/releases/download/v0.1.0-alpha.7/addon-components.yaml
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

vault kv put secret/demo_secrets tokenA=too_many_secrets tokenB=not_enough_secrets

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

# ArgoCD
## Install

```
helm repo add argo https://argoproj.github.io/argo-helm
helm install argocd argo/argo-cd -n argocd --create-namespace --set configs.params."server\.insecure"=true


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

## RBAC
Argo needs privileges to create CAPI Clusters

```
kubectl create clusterrolebinding argocd-application-controller-cluster-admin-binding --clusterrole=cluster-admin --serviceaccount=argocd:argocd-application-controller
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




