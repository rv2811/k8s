# Setup Kubernetes

There are different methods, platforms can be used to setup _Kubernetes_, there are ready tools available in marketplace to quick setup _Kubernetes_ like Kind, K3s and such. those can be used to quick setup and for testing or POC Purpose. these tools are can not be used for Development or Production loads as they do not offer much flexibility or feature to upgrade or much customization to troubleshoot or fix issues. 

We can setup _Self Manageed_ _Kubernetes_ setup for development and production purpose we can setup such environment on virtual machine hosted on any platforms like, Hyper-v, VMware, Vgrant or either hosted in Cloud like Azure, AWS or GCP. which is self managed and can fully customizable. 

We can also use managed services offered by Azure (AKS). AWS (EKS), or Google Cloud (GKS) which is cloud manged service and used to host container nodes.

![Kubernets-Options](https://github.com/user-attachments/assets/cd845fa7-43f5-490c-ab6e-7786b5b3b846)

#### Setup Self Managed Kubernetes

This process includes following setups to setup _Kubernetes_ Self Managed environment
![Kubernets_Steps](https://github.com/user-attachments/assets/b508338d-1eb1-49af-b002-7269e1b64a4d)

##### Setup Step by step

>[!NOTES]
>Our current seutp includes two workner nodes.

1. Create Three VMs (1 for Master Node, 2 for Worker Nodes)
2. Notedown Local IP Address for all three VMs
3. Setup Hostname for VMs using below command
```
sudo hostnamectl set-hostname <host name>
```
_Example_
```
sudo hostnamectl set-hostname kube-master
sudo hostnamectl set-hostname kube-worker1
sudo hostnamectl set-hostname kube-worker2
```
4.  Edit /etc/hosts file and host entry for all three servers to resolve locally
```
sudo vi /etc/hosts
```
```
10.0.2.6	kube-master	kube-master.ap.local
10.0.2.5	kube-worker1 kube-worker1.ap.local
10.0.2.7	kube-worker2 kube-worker2.ap.local
```
5. Install latest updates and upgrade linux kernel to latest version and reboot once done.
```
sudp apt update && sudo apt upgrade
```
6. Disable Swap using below command
```
swapoff -a
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```
7. Forward IPv4 and iptable to see bridged traffic
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

# Verify that the br_netfilter, overlay modules are loaded by running the following commands:
lsmod | grep br_netfilter
lsmod | grep overlay

# Verify that the net.bridge.bridge-nf-call-iptables, net.bridge.bridge-nf-call-ip6tables, and net.ipv4.ip_forward system variables are set to 1 in your sysctl config by running the following command:
sysctl net.bridge.bridge-nf-call-iptables net.bridge.bridge-nf-call-ip6tables net.ipv4.ip_forward
```
8. Install Continer Runtime. (Here we are using containerd as runtime, there are other available like docker)
```
curl -LO https://github.com/containerd/containerd/releases/download/v1.7.14/containerd-1.7.14-linux-amd64.tar.gz
sudo tar Cxzvf /usr/local containerd-1.7.14-linux-amd64.tar.gz
curl -LO https://raw.githubusercontent.com/containerd/containerd/main/containerd.service
sudo mkdir -p /usr/local/lib/systemd/system/
sudo mv containerd.service /usr/local/lib/systemd/system/
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
sudo systemctl daemon-reload
sudo systemctl enable --now containerd

# Check that containerd service is up and running
systemctl status containerd
```
9.  Install runc
```
curl -LO https://github.com/opencontainers/runc/releases/download/v1.1.12/runc.amd64
sudo install -m 755 runc.amd64 /usr/local/sbin/runc
```
10.  Install CNI Plugin
```
curl -LO https://github.com/containernetworking/plugins/releases/download/v1.5.0/cni-plugins-linux-amd64-v1.5.0.tgz
sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.5.0.tgz
```
11.  Install kubeadm, kubelet and kubectl
```
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl gpg

curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.29/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.29/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update
sudo apt-get install -y kubelet=1.29.6-1.1 kubeadm=1.29.6-1.1 kubectl=1.29.6-1.1 --allow-downgrades --allow-change-held-packages
sudo apt-mark hold kubelet kubeadm kubectl

kubeadm version
kubelet --version
kubectl version --client
```
12.  configure crictl to work with containerd
```
sudo crictl config runtime-endpoint unix:///var/run/containerd/containerd.sock
```
>[!IMPORTANT]
>Below steps 13,14,15 to run into Control Plane role only to initialize master node
13.  Initialize Control Plane
```
sudo kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=<Public or LAN IP for Msater Node>--node-name <Name for Master Node>
```
14.  Prepare _kubeconfig_
```
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```
15.  Install Calico
```
kubectl create -f https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/tigera-operator.yaml

curl https://raw.githubusercontent.com/projectcalico/calico/v3.28.0/manifests/custom-resources.yaml -O

kubectl apply -f custom-resources.yaml
```

>[!IMPORTANT]
>Below steps on Cluster node to join master node

16.  Run the command generated in setup 13 which is similer to below.
```
sudo kubeadm join 172.31.71.210:6443 --token xxxxx --discovery-token-ca-cert-hash sha256:xxx
```
Incase to generate joining token use below command on master node.
```
kubeadm token create --print-join-command
```



     







