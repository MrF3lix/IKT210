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




## New Steps


- Reset Cluster
- Install cluster again
- Setup Arogcd
- Get Default credentails: bL8HwaEwc3cx1soD
- Add sample application
- Change service to NodePort: `kubectl patch svc argocd-server -n argocd --type='json' -p '[{"op":"replace","path":"/spec/type","value":"NodePort"}]'`
- Get port with `kubectl get svc -n argocd`
- Access via https://10.225.151.33:31284/

`
### Kustomize Argocd

- Create Repository in gitlab
- Add yaml manifests
- Push
- Create a new definition in argocd

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argocd
spec:
  destination:
    name: ''
    namespace: argocd
    server: 'https://kubernetes.default.svc'
  source:
    path: lab04-argocd/kustomize
    repoURL: >-
      git@gitlab.internal.uia.no:ikt210-g-22h-project2/LabGroup13/lab04-argocd.git
    targetRevision: HEAD
  project: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions: []
```