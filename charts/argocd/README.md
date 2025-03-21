# Useful commands

## Kubernetes
### Interogate cluster
```
kubectl get svc -n argocd
kubectl get pods -n argocd
kubectl describe pod <podname> -n argocd
```

### Restart deployment
```
kubectl rollout restart deployment argocd-server -n argocd
```

##  Helm
### Install chart
```
helm repo add argo-cd https://argoproj.github.io/argo-helm
helm install argocd charts/argocd/ --namespace argocd --create-namespace
```

### Update chart from helm
```
helm upgrade argocd charts/argocd/ -n argocd
```
