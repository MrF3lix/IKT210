## Steps


### Install Argocd


- Install Argocd
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/core-install.yaml
```


- Make Argocd accessible from outside the cluster

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "Nodeport"}}'
```

```bash
kubectl port-forward svc/argocd-repo-server -n argocd 8080:443
```