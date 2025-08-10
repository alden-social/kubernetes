# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/jammy64"
  config.vm.synced_folder ".", "/vagrant", type: "virtualbox"

  COMMON_BOOTSTRAP = <<-SHELL
    set -eux
    swapoff -a && sed -i.bak '/ swap / s/^/#/' /etc/fstab
    modprobe overlay && modprobe br_netfilter
    cat <<EOF | tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF
    cat <<EOF | tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
EOF
    sysctl --system

    apt-get update
    apt-get install -y apt-transport-https ca-certificates curl gnupg lsb-release

    # containerd
    curl -fsSL https://download.docker.com/linux/ubuntu/gpg | gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" \
      | tee /etc/apt/sources.list.d/docker.list > /dev/null
    apt-get update && apt-get install -y containerd.io
    mkdir -p /etc/containerd
    containerd config default | tee /etc/containerd/config.toml >/dev/null
    sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
    systemctl enable --now containerd

    # Kubernetes
    curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | gpg --dearmor -o /usr/share/keyrings/kubernetes-archive-keyring.gpg
    echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /" \
      | tee /etc/apt/sources.list.d/kubernetes.list
    apt-get update && apt-get install -y kubelet kubeadm kubectl
    apt-mark hold kubelet kubeadm kubectl
    systemctl enable --now kubelet
  SHELL

  # Control plane
  config.vm.define "cp" do |node|
    node.vm.hostname = "k8s-cp"
    node.vm.network :private_network, ip: "192.168.56.10"
    node.vm.provider :virtualbox do |vb|
      vb.name = "k8s-cp"
      vb.memory = 3072
      vb.cpus = 2
    end
    node.vm.provision "shell", inline: COMMON_BOOTSTRAP
    node.vm.provision "shell", inline: <<-SHELL
      set -eux
      if [ ! -f /vagrant/.cp-initialized ]; then
        kubeadm init --pod-network-cidr=192.168.0.0/16 --apiserver-advertise-address=192.168.56.10
        mkdir -p $HOME/.kube
        cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
        chown $(id -u):$(id -g) $HOME/.kube/config
        # Calico CNI
        kubectl apply -f https://raw.githubusercontent.com/projectcalico/calico/v3.27.3/manifests/calico.yaml
        # Publish join command for workers
        kubeadm token create --print-join-command > /vagrant/join.sh
        chmod +x /vagrant/join.sh
        touch /vagrant/.cp-initialized
      fi
    SHELL
  end

  # Workers
  [
    {name: "w1", host: "k8s-w1", ip: "192.168.56.11"},
    {name: "w2", host: "k8s-w2", ip: "192.168.56.12"}
  ].each do |w|
    config.vm.define w[:name] do |node|
      node.vm.hostname = w[:host]
      node.vm.network :private_network, ip: w[:ip]
      node.vm.provider :virtualbox do |vb|
        vb.name = w[:host]
        vb.memory = 2048
        vb.cpus = 2
      end
      node.vm.provision "shell", inline: COMMON_BOOTSTRAP
      node.vm.provision "shell", inline: <<-SHELL
        set -eux
        # Wait for join script
        for i in $(seq 1 30); do
          if [ -f /vagrant/join.sh ]; then
            bash /vagrant/join.sh
            exit 0
          fi
          sleep 10
        done
        echo "join.sh not found; ensure control-plane is up first" >&2
        exit 1
      SHELL
    end
  end
end
