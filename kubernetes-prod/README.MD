# Install k8s cluster via kubeadm

sudo apt install -y apt-transport-https ca-certificates curl gpg
VER=1.31
curl -fsSL https://pkgs.k8s.io/core:/stable:/v${VER}/deb/Release.key | sudo echo "deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v${VER}/deb/ /" | sudo tee /etc/apt/sources.list.d/kubernetes.list gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
sudo apt update
sudo apt install wget curl vim git kubelet kubeadm kubectl -y
sudo apt-mark hold kubelet kubeadm kubectl
apt-mark unhold

## Turn off swap
sudo swapoff -a 
sudo sed -i '/ swap / s/^/#/' /etc/fstab

## Enable kernel modules
sudo modprobe overlay
sudo modprobe br_netfilter

## Add some settings to sysctl
sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

## Reload sysctl
sudo sysctl --system

sudo tee /etc/modules-load.d/k8s.conf <<EOF
overlay
br_netfilter
EOF

## Load at runtime
sudo modprobe overlay
sudo modprobe br_netfilter

## Ensure sysctl params are set
sudo tee /etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

## Reload configs
sudo sysctl --system

## Install required packages
sudo apt install -y curl gnupg2 software-properties-common apt-transport-https ca-certificates

## Add Docker repo
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/trusted.gpg.d/docker-archive-keyring.gpg
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"

## Installing Containerd
sudo apt update
sudo apt install -y containerd.io
## Configure containerd and start service
sudo mkdir -p /etc/containerd
sudo containerd config default|sudo tee /etc/containerd/config.toml
## restart containerd
sudo systemctl restart containerd
sudo systemctl enable containerd
systemctl status containerd

sudo systemctl enable kubelet
sudo kubeadm config images pull
sudo kubeadm config images pull --cri-socket /run/containerd/containerd.sock

sudo echo "172.29.20.5 k8s-cluster.otus.loc" >> /etc/hosts
sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --upload-certs \
  --control-plane-endpoint=k8s-cluster.otus.loc

sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16

sudo kubeadm init \
  --pod-network-cidr=10.244.0.0/16 \
  --cri-socket /run/containerd/containerd.sock \
  --upload-certs \
  --control-plane-endpoint=k8s-cluster.otus.loc

 mkdir -p $HOME/.kube
 sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
 sudo chown $(id -u):$(id -g) $HOME/.kube/config

## Join 1 control-plane node
kubeadm join k8s-cluster.otus.loc:6443 --token sr4414.1kvat0palh5o4ik \
    --discovery-token-ca-cert-hash sha256:1d2c7d5a14c5d717a676291ff5ac25e041386e6371d56bb5cc82d5907fdf62a1 \
    --control-plane 
## Join 3 worker nodes
kubeadm join k8s-cluster.computingforgeeks.com:6443 --token sr4l2l.2kvot0pfalh5o4ik \
    --discovery-token-ca-cert-hash sha256:c692fb047e15883b575bd6710779dc2c5af8073f7cab460abd181fd3ddb29a18

kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

## Update control-plane
sudo apt-get update && sudo apt-get install -y --allow-change-held-packages kubeadm=1.32
sudo kubeadm upgrade plan
sudo kubeadm upgrade apply v1.32.0
sudo apt-get update && sudo apt-get install -y --allow-change-held-packages kubelet=1.32 kubectl=1.32.0-00
sudo systemctl daemon-reload
sudo systemctl restart kubelet

## Update worker1
kubectl drain worker1 --ignore-daemonsets --delete-emptydir-data
sudo apt-get update && sudo apt-get install -y --allow-change-held-packages kubeadm=1.32
sudo kubeadm upgrade node
sudo apt-get update && sudo apt-get install -y --allow-change-held-packages kubelet=1.32 kubectl=1.32
sudo systemctl daemon-reload
sudo systemctl restart kubelet
kubectl uncordon worker1 

## Update worker2
kubectl drain worker2 --ignore-daemonsets --delete-emptydir-data
sudo apt-get update && sudo apt-get install -y --allow-change-held-packages kubeadm=1.32
sudo kubeadm upgrade node
sudo apt-get update && sudo apt-get install -y --allow-change-held-packages kubelet=1.32 kubectl=1.32
sudo systemctl daemon-reload
sudo systemctl restart kubelet
kubectl uncordon worker2

## Update worker3
kubectl drain worker3 --ignore-daemonsets --delete-emptydir-data
sudo apt-get update && sudo apt-get install -y --allow-change-held-packages kubeadm=1.32
sudo kubeadm upgrade node
sudo apt-get update && sudo apt-get install -y --allow-change-held-packages kubelet=1.32 kubectl=1.32
sudo systemctl daemon-reload
sudo systemctl restart kubelet
kubectl uncordon worker3

## kubectl get nodes
- cp1       Ready    control-plane   1h   v1.32.1
- worker1   Ready    <none>          1h   v1.32.1
- worker2   Ready    <none>          1h   v1.32.1
- worker3   Ready    <none>          1h   v1.32.1
