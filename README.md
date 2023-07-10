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




