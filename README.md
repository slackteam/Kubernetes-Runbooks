##################### HOST CONFIG ####################### 

sudo setenforce 0

sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config




##################### CONTROL PLANE ONLY ####################### 

sudo firewall-cmd --permanent --add-port=6443/tcp

sudo firewall-cmd --permanent --add-port=10250/tcp

sudo firewall-cmd --permanent --add-port=2379/tcp

sudo firewall-cmd --permanent --add-port=2380/tcp

sudo firewall-cmd --permanent --add-port=10259/tcp

sudo firewall-cmd --permanent --add-port=10257/tcp


##################### WORKER NODES ONLY ####################### 

sudo firewall-cmd --permanent --add-port=10250/tcp
sudo firewall-cmd --permanent --add-port=30000-32767/tcp



(((uuidgen ens192)))

sudo vi /etc/sysconfig/network-scripts/ifcfg-ens192

TYPE=Ethernet
PROXY_METHOD=none
BROWSER_ONLY=no
BOOTPROTO=none
DEFROUTE=yes
IPV4_FAILURE_FATAL=no
IPV6INIT=yes
IPV6_AUTOCONF=yes
IPV6_DEFROUTE=yes
IPV6_FAILURE_FATAL=no
IPV6_ADDR_GEN_MODE=eui64
NAME=ens192
UUID=2f4fd84d-d077-4117-8ff6-47e54ddc76c2
DEVICE=ens192
ONBOOT=yes
PREFIX=24
NETMASK=255.255.255.0
IPADDR=192.168.1.145
GATEWAY=192.168.1.1
DNS1=192.168.1.1

sudo systemctl restart NetworkManager

sudo nmcli connection down ens192 && sudo nmcli connection up ens192

sudo hostnamectl set-hostname kube-hq-master-slt1

cat /etc/hosts

sudo echo "192.168.1.124 kube-hq-master-slt1
192.168.1.144 kube-hq-node-slt1
192.168.1.145 kube-hq-node-slt2
192.168.1.146 kube-hq-node-slt3" >> /etc/hosts 

cat /etc/hosts

free -m

swapoff -a

sudo vi /etc/fstab


########################## INSTALL DOCKER ##########################

sudo yum install -y yum-utils

sudo yum-config-manager \
    --add-repo \
    https://download.docker.com/linux/centos/docker-ce.repo
    
sudo yum install docker-ce docker-ce-cli containerd.io docker-compose-plugin

sudo systemctl start docker

sudo systemctl enable docker


######################### INSTALL CRI-DOCKERD #######################

yum install wget -y

wget https://dl.google.com/go/go1.13.4.linux-amd64.tar.gz

sha256sum go1.13.4.linux-amd64.tar.gz

sudo tar -C /usr/local -xf go1.13.4.linux-amd64.tar.gz

vi ~/.bash_profile

PATH=$PATH:/usr/local/go/bin

source ~/.bash_profile


wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.2.0/cri-dockerd-v0.2.0-linux-amd64.tar.gz


tar xvf cri-dockerd-v0.2.0-linux-amd64.tar.gz

sudo mv ./cri-dockerd /usr/local/bin/

cri-dockerd --help



################## INSTALL KUBERNETES  #################

cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF


sudo yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

sudo systemctl enable --now kubelet

unix:///var/run/cri-dockerd.sock

################### CONTROL PLANE ONLY ###################

kubeadm init --apiserver-advertise-address kube-hq-master-slt1 --cri-socket unix:///var/run/cri-dockerd.sock

To start using your cluster, you need to run the following as a regular user:

  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config


################### CILIUM INSTALL ######################

CILIUM_CLI_VERSION=$(curl -s https://raw.githubusercontent.com/cilium/cilium-cli/master/stable.txt)
CLI_ARCH=amd64
if [ "$(uname -m)" = "aarch64" ]; then CLI_ARCH=arm64; fi
curl -L --fail --remote-name-all https://github.com/cilium/cilium-cli/releases/download/${CILIUM_CLI_VERSION}/cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-${CLI_ARCH}.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-${CLI_ARCH}.tar.gz /usr/local/bin
rm cilium-linux-${CLI_ARCH}.tar.gz{,.sha256sum}

cilium install


################### WORKER NODES ###################

rm /etc/containerd/config.toml
systemctl restart containerd

kubeadm join... 
