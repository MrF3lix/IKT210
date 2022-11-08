## Steps


### Install Argocd


- Install Argocd
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/core-install.yaml
```


- Make Argocd accessible from outside the cluster
  
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

- Get admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
```

## Problems

1. `argocd-server` service was not there. Solution: Used the latest version v2.4.15
2.  
3. Could not create a new app inside argocd
```
Unable to create application: application spec for Argocd is invalid: InvalidSpecError: repository not accessible: rpc error: code = Unavailable desc = connection error: desc = "transport: Error while dialing dial tcp: lookup argocd-repo-server: i/o timeout"
```




## New Setps


- Reset Cluster
- Install cluster again
- Setup Arogcd
- Get Default credentails: bL8HwaEwc3cx1soD