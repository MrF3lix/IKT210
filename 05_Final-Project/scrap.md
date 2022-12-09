# Final Project

## VM's

```
+-----------------------------------------+------------------------------------------------------+
| Name                                    | Networks                                             |
+-----------------------------------------+------------------------------------------------------+
| IKT210-G-22H-Lab Group 13-project-node3 | IKT210-G-22H-g13-prod=10.225.151.16, 192.210.13.201  |
| IKT210-G-22H-Lab Group 13-project-node2 | IKT210-G-22H-g13-prod=10.225.150.5, 192.210.13.29    |
| IKT210-G-22H-Lab Group 13-project-node1 | IKT210-G-22H-g13-prod=10.225.149.133, 192.210.13.170 |
+-----------------------------------------+------------------------------------------------------+
```

## Applications

- Frontpager:
  - http://10.225.149.133:31697

- ArgoCD: 
  - https://10.225.149.133:31549/
  - User: admin
  - Password: jpeoYVdQa51oCVCw

- Grafana: 
  - http://10.225.149.133:30349/
  - User: admin
  - Password: 7aS6r4mVjigX6LT

- Prometheus: 
  - http://10.225.149.133:30236
  - User: NONE
  - Password: NONE

- PWPush: 
  - http://10.225.149.133:30783
  - User: NONE
  - Password: NONE

- Kuma: 
  - http://10.225.149.133:30814
  - User: admin
  - Password: K3HnTeGzyfo79itXz2N7
  - Public: http://10.225.149.133:30814/status/public

- Assignment-05-Base: 
  - http://10.225.149.133:30169
  - User: felixmsa@uia.no
  - Password Test123#

- Betauia.net
  - http://10.225.149.133:31375
  - User:
  - Password:

## Decisions

- Use Flannel for networking
  - Easy to install and use
- ArgoCD for application deployment
- Prometheus & Grafana for monitoring => https://github.com/prometheus-operator/kube-prometheus

## Steps 29.11

1. Add SSH certificates to VM's
   ```bash
    ssh-copy-id -i ./.ssh/id_rsa felixmsa@10.225.151.16
    ssh-copy-id -i ./.ssh/id_rsa felixmsa@10.225.150.5
    ssh-copy-id -i ./.ssh/id_rsa felixmsa@10.225.149.133
   ```
2. Add hosts to ssh config file
   ```bash
    Host ikt210-n-3
        HostName 10.225.151.16
        User felixmsa
        IdentitiesOnly yes
        PubKeyAuthentication yes
        IdentityFile ~/.ssh/id_rsa

    Host ikt210-n-2
        HostName 10.225.150.5
        User felixmsa
        IdentitiesOnly yes
        PubKeyAuthentication yes
        IdentityFile ~/.ssh/id_rsa

    Host ikt210-n-1
        HostName 10.225.149.133
        User felixmsa
        IdentitiesOnly yes
        PubKeyAuthentication yes
        IdentityFile ~/.ssh/id_rsa
   ```
3. Check if swap devices exist and disable them
   ```bash
    cat /proc/swaps
    swapoff -a
   ```
3. Make sure all nodes have different mac addresses
   ```bash
    cat /sys/class/dmi/id/product_uuid
   ```
4. Install container runtime `containerd`
   ```bash
    apt-get update
    apt-get install containerd
   ```
5. Enable configuration
   ```bash
    mkdir /etc/containerd/ && containerd config default > /etc/containerd/config.toml
   ```
6. Enable `SystemdCgroup` driver
   ```
    plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
        ...
        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            SystemdCgroup = true
   ```
7. Restart `containerd`
   ```bash
    systemctl restart containerd
   ```
8. IP Forwarding
   ```bash
    cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
    overlay
    br_netfilter
    EOF
    sudo modprobe overlay
    sudo modprobe br_netfilter
    # sysctl params required by setup, params persist across reboots
    cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
    net.bridge.bridge-nf-call-iptables = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    net.ipv4.ip_forward = 1
    EOF
    # Apply sysctl params without reboot
    sudo sysctl --system
   ```
9. Install `kubelet`, `kubeadm` and `kubectl`
    ```bash
    apt-get install -y apt-transport-https ca-certificates curl
    curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

    echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

    apt-get update
    apt-get install -y kubelet kubeadm kubectl
    apt-mark hold kubelet kubeadm kubectl
    ```
10. Init cluster: **Note:** `10.244.0.0/16` is used for the pod network as specified in https://github.com/flannel-io/flannel/blob/master/Documentation/kubernetes.md
    ```bash
    kubeadm init --pod-network-cidr=10.244.0.0/16 --service-cidr=10.97.0.0/16 --control-plane-endpoint=10.225.149.133
    ```
11. Join Nodes
    ```bash
    kubeadm join 10.225.149.133:6443 --token z8aj4m.7gw51ul1lkjir7fp \
	--discovery-token-ca-cert-hash sha256:a57da44703cdcd00d72a524d001913ed93e1fed668d21e803d623cd70dc68a3c
    ```
12. Copy access configuration and add to local configuration
    ```bash
    cat /etc/kubernetes/admin.conf
    ```
13. Verify that nodes are joined
    ```bash
    kubectl get nodes

    NAME                                      STATUS     ROLES           AGE     VERSION
    ikt210-g-22h-lab-group-13-project-node1   NotReady   control-plane   5m14s   v1.25.4
    ikt210-g-22h-lab-group-13-project-node2   NotReady   <none>          4m25s   v1.25.4
    ikt210-g-22h-lab-group-13-project-node3   NotReady   <none>          4m24s   v1.25.4
    ```
14. Install Flannel
    ```bash
    kubectl apply -f https://raw.githubusercontent.com/flannel-io/flannel/master/Documentation/kube-flannel.yml
    ```

15. Install Argocd
    ```bash
    kubectl create namespace argocd
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.5.2/manifests/install.yaml
    ```
16. Enable NodePort for Argocd
    ```bash
    kubectl patch svc argocd-server -n argocd --type='json' -p '[{"op":"replace","path":"/spec/type","value":"NodePort"}]'
    ```
17. Get Admin Password
    ```bash
    kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo
    ```
18. Access Argocd via https://10.225.149.133:31549/

19. Create Argocd application in Argocd

    ```yaml
    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
    name: argocd
    spec:
        project: default
        source:
            repoURL: 'git@gitlab.internal.uia.no:ikt210-g-22h-project2/LabGroup13/argocd.git'
            path: kustomize
            targetRevision: HEAD
        destination:
            server: 'https://kubernetes.default.svc'
            namespace: argocd
        syncPolicy:
            automated:
                prune: true
                selfHeal: true
    ```

20. Create storage class
    ```yaml
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
    name: default-storage
    provisioner: kubernetes.io/no-provisioner
    volumeBindingMode: WaitForFirstConsumer
    ```
21. Set as default
    ```bash
    kubectl patch sc default-storage -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'
    ```

22. Create monitoring repository https://gitlab.internal.uia.no/ikt210-g-22h-project2/LabGroup13/monitoring. Use https://github.com/prometheus-operator/kube-prometheus as a base.

23. Add Monitoring to argocd

    ```yaml
    project: default
    source:
    repoURL: 'git@gitlab.internal.uia.no:ikt210-g-22h-project2/LabGroup13/monitoring.git'
    path: .
    targetRevision: HEAD
    destination:
    server: 'https://kubernetes.default.svc'
    namespace: monitoring
    syncPolicy:
    automated:
        prune: true
        selfHeal: true
    ```

24. Enable access to grafana using NodePort
    ```bash
    kubectl patch svc grafana -n monitoring --type='json' -p '[{"op":"replace","path":"/spec/type","value":"NodePort"}]'
    ```
25. Log into grafana and change default admin password to: 7aS6r4mVjigX6LT
26. Compute Resources Dashboard: http://10.225.149.133:30349/d/efa86fd1d0c121a26444b636a3f509a8/kubernetes-compute-resources-cluster?orgId=1&refresh=10s


## Steps 6.12

1. Create Repository for PwPush Deployment files https://gitlab.internal.uia.no/ikt210-g-22h-project2/LabGroup13/password-pusher-deployment
2. Add Deployment File
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
    name: pwpush
    spec:
    replicas: 1
    selector:
        matchLabels:
        app: pwpush
    template:
        metadata:
        labels:
            app: pwpush
        spec:
        containers:
            - name: pwpush
            image: pglombardo/pwpush-ephemeral:release
            ports:
            - containerPort: 5100
    ```
3. Add Service File
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
    name: pwpush
    spec:
    type: NodePort
    selector:
        app: pwpush
    ports:
        - port: 5100
        targetPort: 5100
    ```
4. Add to ArgoCD
5. Test Page: http://10.225.149.133:30783/
6. Create Repository for Uptime Kuma Deployment files https://gitlab.internal.uia.no/ikt210-g-22h-project2/LabGroup13/uptime-kuma-deployment
7. Add Deployment File
    ```yaml
    apiVersion: apps/v1
    kind: Deployment
    metadata:
    name: uptime-kuma
    spec:
    replicas: 1
    selector:
        matchLabels:
        app: uptime-kuma
    template:
        metadata:
        labels:
            app: uptime-kuma
        spec:
        containers:
            - name: uptime-kuma
            image: louislam/uptime-kuma:1.18.5-alpine
            ports:
            - containerPort: 3001
    ```
8. Add Service File
    ```yaml
    apiVersion: v1
    kind: Service
    metadata:
    name: uptime-kuma
    spec:
    type: NodePort
    selector:
        app: uptime-kuma
    ports:
        - port: 3001
        targetPort: 3001
    ```
9. Add to ArgoCD
10. Test Page: http://10.225.149.133:30814/
11. Create Account Kuma
12. Login to Kuma
13. Add Monitor for ArgoCD, Grafana, Password Pusher, Prometheus

## Steps 6.12 - Assignment-05-base Application

1. Create repository in gitlab https://gitlab.internal.uia.no/ikt210-g-22h-project2/LabGroup13/assignment-05-base/
1. Create dockerfile
    ```dockerfile
    FROM mcr.microsoft.com/dotnet/sdk:5.0 AS build-env
    WORKDIR /app

    COPY . ./
    RUN dotnet restore
    RUN dotnet publish -c Debug -o out

    FROM mcr.microsoft.com/dotnet/aspnet:5.0
    WORKDIR /app
    COPY --from=build-env /app/out .
    EXPOSE 80
    ENTRYPOINT ["dotnet", "Site.dll"]
    ```
2. Login to docker registry
    ```bash
    docker login registry.internal.uia.no --username felixmsa
    ```
3. Build docker image
    ```bash
    docker build . -t registry.internal.uia.no/ikt210-g-22h-project2/labgroup13/assignment-05-base:latest
    docker buildx build --platform linux/amd64 . -t registry.internal.uia.no/ikt210-g-22h-project2/labgroup13/assignment-05-base:latest
    ```
4. Push image to docker registry
    ```bash
    docker push registry.internal.uia.no/ikt210-g-22h-project2/labgroup13/assignment-05-base:latest
    ```
5. Add docker registry authentication to cluster
   1. Create Access Token
   2. Create secret
    ```bash
    kubectl create secret docker-registry regcred --docker-server=registry.internal.uia.no --docker-username=felixmsa --docker-password=glpat-JRvuGjZqsatSyfUsjD4L --docker-email=felixmsa@uia.no -n assignment-05
    ```
6. NOT WORKING: Create Gitlab CI to automatically push to registry when application changes
7. NOT WORKING: Update kustomize image version after push to registry to update cluster
8. Create Kustomize configuration => https://gitlab.internal.uia.no/ikt210-g-22h-project2/LabGroup13/assignment-05-base/-/tree/master/kustomize
9.  Add init container to wait for database start
    ```yaml
    initContainers:
    - name: postgres-listener
        image: postgres:15.1
        imagePullPolicy: IfNotPresent
        command: [
            "sh", "-c",
            "until pg_isready -h database -p 5432; do echo waiting for database; sleep 2; done;"
            ]
    ```

TODO:
- Store Database password in another config
- Use PVC for database so it is actually persistent


## Steps 6.12 - Betauia.net

1. Create separate repository in gitlab
1. Build docker images and push to gitlab registry
    ```bash

    docker buildx build --platform linux/amd64 ./betauia -t registry.internal.uia.no/ikt210-g-22h-project2/labgroup13/betauia.net/backend:latest
    docker push registry.internal.uia.no/ikt210-g-22h-project2/labgroup13/betauia.net/backend:latest

    docker buildx build --platform linux/amd64 ./frontend -t registry.internal.uia.no/ikt210-g-22h-project2/labgroup13/betauia.net/frontend:latest
    docker push registry.internal.uia.no/ikt210-g-22h-project2/labgroup13/betauia.net/frontend:latest

    ```
2. Create secret to pull images correctly
    ```bash
    kubectl create secret docker-registry regcred --docker-server=registry.internal.uia.no --docker-username=felixmsa --docker-password=glpat-JRvuGjZqsatSyfUsjD4L --docker-email=felixmsa@uia.no -n assignment-05
    ```
3. Create Kustomize files (see gitlab repo)
4. Add to argocd

## Problems

1. We first choose to use cilium for the networking. Unfortunately it didn't work as expected and to make it work we would need to recompile the kernel and enable BPF: https://docs.cilium.io/en/v1.12/operations/system_requirements/#admin-system-reqs.
2. Because we tried to use cilium the VM's had a network interface configured `cilium_vxlan` and `cilium_net`. While installing Flannel we had a problem assigning IP addresses because there were the previously configured interfaces. To fix this we ran `ip link delete cilium_vxlan` and `ip link delete cilium_net`
3. Cannot create argocd application within argocd. Error message: Unable to create application: error creating application: the server could not find the requested resource (post applications.argoproj.io)


## Problem 6.12

1. Couldn't access gitlab docker registry. => Create Secret in cluster `kubectl create secret docker-registry` => Replace Docker with kaniko
2. Docker images were build on an amd64 platform. Using `docker buildx build --platform linux/amd64` fixed the problem. 
3. Database is not persistent => Create PV and PVC for database
4. Could not run the docker build for the betauia.net frontend, `yarn install` failes during image building. => Use CI Runner to build the application

# Problem 8.12

1. Dockerbuild doesn't work on shared gitlab ci runners. => Replace Docker with kaniko

## TODO 8.12

- [x] Assignment-05-base: Postgres Persistence => PVC
- [ ] Assignment-05-base: Update Dependencies
- [x] Assignment-05-base: Setup CI to push images and update argocd
- [x] Assignment-05-base: Use secrets to store db user and password
- [x] betauia: Redis Persistence => PVC
- [ ] betauia: Update Dependencies
- [x] betauia: Fix yarn build
- [x] betauia: Publish frontend docker image
- [x] betauia: Setup CI to push images and update argocd
- [x] betauia: Use secrets to store db user and password
- [ ] NSA: Read hardening guide and define what to do


## Problem 9.12


1. Database initi doesn't work. => No Further Steps to fix it
```
Unhandled Exception: System.AggregateException: One or more errors occurred. (The given key '17736' was not present in the dictionary.) ---> System.Collections.Generic.KeyNotFoundException: The given key '17736' was not present in the dictionary.
```

## Further Improvements


- Use version number instead of the latest tag on the docker images
- Update version number in kustomzie file after a new docker image is pushed to update the cluster automatically
- Betauia: Fix DB initialization in code
- Betauia: Update dependencies
- Assignment-05-Base: Update dependencies
- Create Network Policies
- 