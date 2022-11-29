# Final Project

## VM's

+-----------------------------------------+------------------------------------------------------+
| Name                                    | Networks                                             |
+-----------------------------------------+------------------------------------------------------+
| IKT210-G-22H-Lab Group 13-project-node3 | IKT210-G-22H-g13-prod=10.225.151.16, 192.210.13.201  |
| IKT210-G-22H-Lab Group 13-project-node2 | IKT210-G-22H-g13-prod=10.225.150.5, 192.210.13.29    |
| IKT210-G-22H-Lab Group 13-project-node1 | IKT210-G-22H-g13-prod=10.225.149.133, 192.210.13.170 |
+-----------------------------------------+------------------------------------------------------+

## Decisions

- Use Flannel for networking
  - Easy to install and use
- ArgoCD for application deployment
- Prometheus & Grafana for monitoring => https://github.com/prometheus-operator/kube-prometheus

## Steps

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
18. Access Argocd via https://10.225.149.133:32213/

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
27. 


## Problems

1. We first choose to use cilium for the networking. Unfortunately it didn't work as expected and to make it work we would need to recompile the kernel and enable BPF: https://docs.cilium.io/en/v1.12/operations/system_requirements/#admin-system-reqs.
2. Because we tried to use cilium the VM's had a network interface configured `cilium_vxlan` and `cilium_net`. While installing Flannel we had a problem assigning IP addresses because there were the previously configured interfaces. To fix this we ran `ip link delete cilium_vxlan` and `ip link delete cilium_net`
3. Cannot create argocd application within argocd. Error message: Unable to create application: error creating application: the server could not find the requested resource (post applications.argoproj.io)