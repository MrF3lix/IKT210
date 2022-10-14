## Steps


### Storage Classes

- One storage class exists. This class was created during the kubernetes setup as a test.

```bash
kubectl get sc

NAME         PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
local-path   rancher.io/local-path   Delete          WaitForFirstConsumer   false                  27d
```

- Create a new default storage class

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: default-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```

- Set as default

```bash
kubectl patch sc default-storage -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
```

- Check if it exists

```bash
kubectl get sc

NAME                        PROVISIONER                    RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
default-storage (default)   kubernetes.io/no-provisioner   Delete          WaitForFirstConsumer   false                  2m47s
local-path                  rancher.io/local-path          Delete          WaitForFirstConsumer   false                  27d
```

### Ghost

1. Create deployment file
2. Create service file
3. Add Proxy already

```bash
kubectl apply -f ./deployment.yaml
kubectl apply -f ./service.yaml

kubectl port-forward service/nginx 3000:80 -n lab03
```

4. Create PVC

### Database

1. Setup Database deployment


### Connection


### Reverse Proxy




### Problems

- Added a liveness Probe with an initial delay that was too short, the migrations couldn't run in the 3 seconds we configured so it stopped. When the container restarted the db migrations failed because the previous wasn't completed