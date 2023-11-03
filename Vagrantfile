# -*- mode: ruby -*-
# vi: set ft=ruby :

NUMBERWORKERS=4
CRIOVERSION="1.7.8"
RUNCVERSION="1.1.10"
CNIVERSION="1.3.0"
K8SVERSION="1.28.3" #or get latest  from https://dl.k8s.io/release/stable.txt

#=======================================================================================================================

$prepare = <<-'SCRIPT'
sudo apt-get update
sudo apt-get -y dist-upgrade
timedatectl set-timezone Europe/Moscow
sudo apt-get -y install git nano curl wget nmon jq net-tools mc conntrack apt-transport-https ca-certificates

#prepare bridge
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay
sudo modprobe br_netfilter
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF
sudo sysctl --system

#crio_install
curl -LfsS https://github.com/containerd/containerd/releases/download/v${CRIOVERSION}/containerd-${CRIOVERSION}-linux-amd64.tar.gz -o containerd.tar.gz
sudo tar Cxzvf /usr/local containerd.tar.gz
sudo mkdir -pv /usr/local/lib/systemd/system
sudo cp -v /vagrant/Desktop/containerd.service /usr/local/lib/systemd/system/containerd.service
sudo systemctl daemon-reload
sudo systemctl enable containerd
rm -rf containerd.tar.gz
curl -LfsS https://github.com/containernetworking/plugins/releases/download/v${CNIVERSION}/cni-plugins-linux-amd64-v${CNIVERSION}.tgz -o cni.tgz
sudo mkdir -p /opt/cni/bin
sudo tar Cxzvf /opt/cni/bin cni.tgz
rm -rf cni.tgz
sudo mkdir -p /etc/containerd
sudo cp -v /vagrant/Desktop/containerd-config.toml /etc/containerd/config.toml
sudo systemctl restart containerd
curl -LfsSO https://github.com/opencontainers/runc/releases/download/v${RUNCVERSION}/runc.amd64
sudo install -o root -g root -m 755 runc.amd64 /usr/local/sbin/runc
rm -rf runc.amd64

#kubeadm
#K8SVERSION=$(curl -Ls https://dl.k8s.io/release/stable.txt)
curl -fsSL https://pkgs.k8s.io/core:/stable:/$K8SVERSION/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/$K8SVERSION/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl

#kubelet
set -e
if=$1
node_ip=$(ip -4 addr show ${if} | grep "inet" | head -1 | awk '{print $2}' | cut -d/ -f1)
echo "KUBELET_EXTRA_ARGS=--node-ip=${node_ip}" | sudo tee /etc/sysconfig/kubelet
sudo systemctl daemon-reload
sudo systemctl restart kubelet

SCRIPT

Vagrant.configure("2") do |config|
    config.vm.box = "bento/debian-12.1"
    config.vm.box_version = "202309.08.0"
    config.vm.provider "virtualbox" do |vb|
      vb.memory = 2048
      vb.cpus = 2
    end
  
    config.vm.define "master" do |node|
      node.vm.hostname = "master"
      node.vm.network "private_network", ip: "192.168.5.1"
      node.vm.provision "shell", inline: $prepare
      node.vm.provision "initmaster", privileged: true, inline: <<-SHELL
        sudo kubeadm init --apiserver-advertise-address=192.168.5.1 --pod-network-cidr=10.244.0.0/16
        #mkdir -p /vagrant/Desktop/.kubevagrant
        sudo cp -i /etc/kubernetes/admin.conf /vagrant/.kube/configvagrant
        sudo chown $(id -u):$(id -g) /vagrant/.kube/configvagrant
        kubectl create -f /vagrant/tigera-operator.yaml
        kubectl create -f /vagrant/custom-resources.yaml
        echo "openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl dgst -sha256 -hex | sed 's/^.* //'" > /vagrant/keyandhash
        SHELL
        
    end
  
    (1..$NUMBERWORKERS).each do |i|
      config.vm.define "worker-#{i}" do |node|
        node.vm.hostname = "worker-#{i}"
        node.vm.network "private_network", ip: "192.168.5.#{10 + i}"
        node.vm.provision "shell", inline: $prepare
        node.vm.provision "getkey", privileged: true, inline: <<-SHELL
            sudo kubeadm join 192.168.5.1:6443 --token $(cat /vagrant/key) --discovery-token-ca-cert-hash sha256:$(cat /vagrant/hash)
        SHELL
      end
    end


end