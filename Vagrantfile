servers = [
  {
    :name => "master",
    :type => "master",
    :box => "centos/7",
    :ip => "10.0.0.10",
    :RAM => "3072",
    :CPU => "2"
  },
  {
    :name => "worker1",
    :type => "worker",
    :box => "centos/7",
    :ip => "10.0.0.11",
    :RAM => "3072",
    :CPU => "2"
  },
  {
    :name => "worker2",
    :type => "worker",
    :box => "centos/7",
    :ip => "10.0.0.12",
    :RAM => "3072",
    :CPU => "2"
  }
]

Vagrant.configure("2") do |config|
  servers.each do |opts|
    config.vm.synced_folder ".", "/vagrant", type: 'virtualbox'
    config.vm.define opts[:name] do |config|
      config.vm.box = opts[:box]
      config.vm.network "private_network", ip: opts[:ip]
      config.vm.hostname = opts[:name]
      config.vm.provider "virtualbox" do |v|
        v.name = opts[:name]
        v.customize ["modifyvm", :id, "--memory", opts[:RAM]]
        v.customize ["modifyvm", :id, "--cpus", opts[:CPU]]
      end
      config.vm.provision "shell", inline: $provisionAll, env: {"IPADDR" => opts[:ip]}
      if opts[:type] == "master"
        config.vm.provision "shell", inline: $provisionMaster
      else
        config.vm.provision "shell", inline: $provisionNode
      end
    end
  end
end

$provisionAll = <<-SCRIPT
sudo yum update && sudo yum upgrade
echo "Installing Docker"
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install -y docker-ce
systemctl enable --now docker
echo "Install k8s cli tools"
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
EOF
setenforce 0
sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
yum install -y kubelet kubeadm kubectl --disableexcludes=kubernetes
echo  "KUBELET_EXTRA_ARGS=--node-ip=$IPADDR" > /etc/sysconfig/kubelet
systemctl enable --now kubelet
echo "Disable swap"
swapoff -a
sed -i '/ swap / s/^/#/' /etc/fstab
echo "Configure Docker Daemon"
cat > /etc/docker/daemon.json <<EOF
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
EOF
echo "restart Docker"
systemctl daemon-reload
systemctl restart docker
echo "edit iptables view of bridge traffic"
cat <<EOF > /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sysctl --system
SCRIPT

# This script runs kubeadm commands to initialise the cluster, using the master ip
$provisionMaster = <<-SCRIPT
kubeadm init --pod-network-cidr 10.0.0.10/24 --apiserver-advertise-address 10.0.0.10
kubeadm token create --print-join-command > /vagrant/kubeadm_join_cmd.sh
chmod +x /vagrant/kubeadm_join_cmd.sh
cp /etc/kubernetes/admin.conf /vagrant/config
chmod 644 /vagrant/config
mkdir -p ~/.kube && cp /etc/kubernetes/admin.conf ~/.kube/config
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
echo "kubectl --kubeconfig config get nodes"
SCRIPT
$provisionNode = <<-SCRIPT
/vagrant/kubeadm_join_cmd.sh
SCRIPT
