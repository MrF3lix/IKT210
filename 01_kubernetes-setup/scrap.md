1. Check for existing swap devices and files `cat /proc/swaps`
1. Disable swap: `swapoff -a`
1. Verify that each VM has a unique MAC `sudo cat /sys/class/dmi/id/product_uuid`
1. Install container runtime from https://github.com/containerd/containerd/blob/main/docs/getting-started.md
1.1 Install runtime containerd `sudo apt-get update && sudo apt-get install containerd`
1. INstall kubelet kubeadm kubectl

```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl

sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

1. Create config file for containerd `mkdir /etc/containerd/ && containerd config default > /etc/containerd/config.toml`

1. Use systemd cgroup in containerd runtime

```
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
            SystemdCgroup = true
```

1. Restart containerd

1. Install `crictl` and `conntrack`

1. Forwarding IPv4 config

```
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# sysctl params required by setup, params persist across reboots
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# Apply sysctl params without reboot
sudo sysctl --system
```

1. Upgrade `kubeadm` to v1.25.0

```
apt-mark unhold kubeadm && \
apt-get update && apt-get install -y kubeadm=1.25.0-00 && \
apt-mark hold kubeadm
apt-mark hold kubectl
apt-mark hold kubelet

```

1. Apply upgrade `sudo kubeadm upgrade apply v1.25.x`

2. Setup cgroups `sudo kubeadm init --config kubeadm-config.yaml` using the kubeadm-config.yaml file


3. Join nodes

```
kubeadm join 192.168.210.20:6443 --token vw6ieo.744nqb3jkb20ulgn \
	--discovery-token-ca-cert-hash sha256:be0f492311ad78c4ef0c0ff0da72b7fc451ca9e69f21be348f82bdd1bad85f40
```

1. Copy kubectl config to local machine
1. Download calico manifest `curl https://raw.githubusercontent.com/projectcalico/calico/v3.24.1/manifests/calico.yaml -O`
2. Apply manifest `kubectl apply -f calico.yaml`