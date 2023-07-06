# Create NKE Cluster
## MetalLB

# Configure CAPX
https://opendocs.nutanix.com/capx/v1.2.x/getting_started/

```
clusterctl init -i nutanix
```

## CAAPH
```
kubectl apply -f https://github.com/kubernetes-sigs/cluster-api-addon-provider-helm/releases/download/v0.1.0-alpha.6/addon-components.yaml
```

# ArgoCD
## Install
```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.8.0-rc1/manifests/install.yaml
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

Proxy:
```
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

or Patch to use LoadBalancer

```
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

argocd login localhost:8080
(or)
https://localhost:8080/

# Kyverno
## Install
```
kubectl create -f https://github.com/kyverno/kyverno/releases/download/v1.10.1-rc.1/install.yaml
```
## Policy
```
kubectl apply -f scripts/kyverno-policy.yaml
```




