######################
# включаю ip forward #
######################
sudo sysctl -w net.ipv4.ip_forward=1
sudo vim /etc/sysctl.conf

  # Uncomment the next line to enable packet forwarding for IPv4
  net.ipv4.ip_forward=1

sudo sysctl -p
cat /proc/sys/net/ipv4/ip_forward

###############################
# включаю модуль br_netfilter #
###############################
sudo modprobe br_netfilter
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sudo sh -c 'echo "br_netfilter" > /etc/modules-load.d/br_netfilter.conf'

########################
# установка containerd #
########################
sudo apt update
sudo apt install -y containerd
sudo mkdir /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml
sudo vim /etc/containerd/config.toml

[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true

sudo systemctl restart containerd

#################
# установка k8s #
#################
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.31/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.31/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo systemctl enable --now kubelet

# kubeadm-config.yaml
kind: ClusterConfiguration
apiVersion: kubeadm.k8s.io/v1beta4
kubernetesVersion: v1.31.0
networking:
  podSubnet: "10.244.0.0/16" # --pod-network-cidr apparently special value for flannel
  serviceSubnet: "10.255.0.0/16"
---
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: systemd

sudo kubeadm init --config kubeadm-config.yaml

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

#####################
# установка flannel #
#####################
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml


##########################
# джойним ноды в кластер #
##########################
kubeadm join 10.129.0.21:6443 --token ssfww2.2q4gw6j98qls5jlk \
	--discovery-token-ca-cert-hash sha256:4493a4acfa08f55868e60a76e10ec3082dea71053c806c7dcb2604704a3ee452


skuzmin@master-1:~$ kubectl get nodes -o wide
NAME       STATUS   ROLES           AGE   VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
master-1   Ready    control-plane   91m   v1.31.5   10.129.0.21   <none>        Ubuntu 24.04.1 LTS   6.8.0-51-generic   containerd://1.7.12
worker-1   Ready    <none>          75m   v1.31.5   10.129.0.5    <none>        Ubuntu 24.04.1 LTS   6.8.0-51-generic   containerd://1.7.12
worker-2   Ready    <none>          61m   v1.31.5   10.129.0.10   <none>        Ubuntu 24.04.1 LTS   6.8.0-51-generic   containerd://1.7.12
worker-3   Ready    <none>          60m   v1.31.5   10.129.0.28   <none>        Ubuntu 24.04.1 LTS   6.8.0-51-generic   containerd://1.7.12



#######################
# upgrade master node #
#######################

sudo vim /etc/apt/sources.list.d/kubernetes.list

  v1.31 -> v1.32

sudo apt update
sudo apt-cache madison kubeadm

sudo apt-mark unhold kubeadm && \
sudo apt-get update && sudo apt-get install -y kubeadm='1.32.1-1.1' && \
sudo apt-mark hold kubeadm

kubeadm version
sudo kubeadm upgrade plan
sudo kubeadm upgrade apply v1.32.1


kubectl drain master-1 --ignore-daemonsets

sudo apt-mark unhold kubelet kubectl && \
sudo apt-get update && sudo apt-get install -y kubelet='1.32.1-1.1' kubectl='1.32.1-1.1' && \
sudo apt-mark hold kubelet kubectl

sudo systemctl daemon-reload
sudo systemctl restart kubelet

kubectl uncordon master-1

#######################
# upgrade worker node #
#######################

sudo apt-mark unhold kubeadm && \
sudo apt-get update && sudo apt-get install -y kubeadm='1.32.1-1.1' && \
sudo apt-mark hold kubeadm

sudo kubeadm upgrade node

kubectl drain worker-1 --ignore-daemonsets

sudo apt-mark unhold kubelet kubectl && \
sudo apt-get update && sudo apt-get install -y kubelet='1.32.1-1.1' kubectl='1.32.1-1.1' && \
sudo apt-mark hold kubelet kubectl

sudo systemctl daemon-reload
sudo systemctl restart kubelet

kubectl uncordon worker-1

skuzmin@master-1:~$ kubectl get nodes -o wide
NAME       STATUS   ROLES           AGE     VERSION   INTERNAL-IP   EXTERNAL-IP   OS-IMAGE             KERNEL-VERSION     CONTAINER-RUNTIME
master-1   Ready    control-plane   4h36m   v1.32.1   10.129.0.21   <none>        Ubuntu 24.04.1 LTS   6.8.0-51-generic   containerd://1.7.12
worker-1   Ready    <none>          4h20m   v1.32.1   10.129.0.5    <none>        Ubuntu 24.04.1 LTS   6.8.0-51-generic   containerd://1.7.12
worker-2   Ready    <none>          4h6m    v1.32.1   10.129.0.10   <none>        Ubuntu 24.04.1 LTS   6.8.0-51-generic   containerd://1.7.12
worker-3   Ready    <none>          4h5m    v1.32.1   10.129.0.28   <none>        Ubuntu 24.04.1 LTS   6.8.0-51-generic   containerd://1.7.12
