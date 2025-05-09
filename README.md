# k8s-cluster
Require command and tool
- Ansible = 2.x
- helm chart = 3.17
- kubectl = 1.32.x
# Server Inventory
- k8s-control-plan: 172.16.11.2
- k8s-worker-node: 172.16.11.3->5
- bastion-host: 172.16.11.6
- proxy-host: 172.16.11.7
- d tabase-host: 172.16.11.8
- nfs-host: 172.16.11.9
# Installation manually
The installation process typically involves the following steps:
## 1.Prepare the Environment:
Install a supported Linux distribution on all the nodes (master and worker).
- OS: Ubuntu Server 24.04 LTS
- One Master node
- One Worker node
## 1.1 Installation packages
```sh
sudo apt update && sudo apt upgrade -y
```
## 1.2 Remove swap drive
```sh
sudo /sbin/swapon -s
sudo /sbin/swapoff -a
sudo vi /etc/fstab (comment out swap #/swap.img      none    swap    ..)
```
## 1.3 Install containerd
```sh
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
echo \
"deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update && sudo apt-get install containerd.io
```
## 1.4 Enable containerd default configuration
```sh
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo sed -i 's/disabled_plugins = \["cri"\]/disabled_plugins = []/' /etc/containerd/config.toml
```
## 1.5 verify change
```sh
grep 'SystemdCgroup' /etc/containerd/config.toml
```
SystemdCgroup = true
```sh
grep 'disabled_plugins' /etc/containerd/config.toml
```
Disabled_plugins = []
## 1.6 Restart containerd service and enable on start status
```sh
sudo systemctl restart containerd.service
sudo systemctl enable --now containerd
sudo systemctl status containerd
```
## 1.7 Set network iptable
```sh
sudo modprobe br_netfilter
sudo -i
echo 1 > /proc/sys/net/bridge/bridge-nf-call-iptables
echo 1 > /proc/sys/net/ipv4/ip_forward
exit
```
## 2. Install Kubernetes Components:
Install the Kubernetes control plane components (kube-apiserver, kube-controller-manager, kube-scheduler, etcd) on the master node.
Install the kubelet and the container runtime on each worker node.
Configure the kubelet to connect to the master node.
## 2.1 Add kubernetes repository
```sh
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.33/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.33/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
## 2.2 Install kubernetes packages && Disable from auto update
```sh
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl nfs-common
sudo apt-mark hold kubelet kubeadm kubectl
```
## 2.3 Update Timezone 
```sh
sudo timedatectl set-timezone Asia/Phnom_Penh
cat <<EOF | sudo tee /etc/crictl.yaml
runtime-endpoint: unix:///var/run/containerd/containerd.sock
image-endpoint: unix:///var/run/containerd/containerd.sock
timeout: 2
debug: true
pull-image-on-create: false
EOF
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
EOF
```
==>Repeat from step 2 for all worker nodes
## 3. Install Kubernetes Components:
- Initialize the Kubernetes cluster on the Master node.
- NOTE: Update --pod-network-cidr=192.168.0.0/16 to match with your environment
 ```sh
 sudo kubeadm init --control-plane-endpoint --pod-network-cidr=192.168.0.0/16
```
 ```sh
mkdir ~/.kube
sudo cp /etc/kubernetes/admin.conf ~/.kube/config
sudo chown ${USER}:${USER} ~/.kube/config
```
```sh
kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.30.0/manifests/calico.yaml
```
## 3.1 Join the worker nodes to the cluster.
```sh
sudo kubeadm join <master-ip>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```
## 3.4 Check that all nodes have joined (On master node):
```sh
kubectl get nodes
```

























