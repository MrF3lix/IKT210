kubectl -n kube-system get pods -l k8s-app=kube-dns --no-headers -o custom-columns=NAME:.metadata.name,HOSTIP:.status.hostIP | while read pod host; do echo "Pod ${pod} on host ${host}"; kubectl -n kube-system exec $pod -c kubedns -- cat /etc/resolv.conf; done


kubectl exec dnsutils -- nslookup kubernetes.default
  ;; connection timed out; no servers could be reached
  command terminated with exit code 1
  
  
kubectl exec dnsutils -- nslookup kubernetes.default 10.96.0.10

kubectl exec dnsutils -- cat /etc/resolv.conf
  nameserver 10.96.0.10
  options ndots:5
  
  
kubectl -n kube-system edit configmap coredns


kubeadm config print init-defaults
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
  
  
  
kubectl cluster-info dump | grep -m 1 service-cluster-ip-range
 --service-cluster-ip-range=10.96.0.0/12
 
 
kubectl cluster-info dump | grep -m 1 pod-network-cidr
 NONE
 
 
 
 
 
 SET POD CIDR TO 192.168.0.0/16
 
 
 
 
 
 
sudo kubeadm reset

kubeadm init --pod-network-cidr=192.168.0.0/16 --service-cidr=10.97.0.0/16 --control-plane-endpoint=10.225.151.33

kubeadm join 10.225.151.33:6443 --token 8rh26g.s5mbvnohzmkm8te4 \
	--discovery-token-ca-cert-hash sha256:4670674b3a6cb06b833c90cdc0e910dbfb4c0b4123f58fc4f5dee5c58a2a38ba
	
	
kubectl patch svc argocd-server -n argocd --type='json' -p '[{"op":"replace","path":"/spec/type","value":"NodePort"}]'
