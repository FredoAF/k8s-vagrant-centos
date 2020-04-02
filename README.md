# k8s-vagrant-centos
- This is forked from https://github.com/RobboF/k8s-vagrant-centos which used a custom box and is setup for production
- This creates a single manager and 2 worker nodes for testing / development
- Standard centos7 boxes are used and configured to meet the pre-reqs
- Kubeadm is used to create cluster, and then workers joined automatically
- to run: `vagrant up`
- Then copy the last command to run kubectl commands against the cluster: `kubectl --kubeconfig config get nodes`

## troubleshooting
- the default centos7 doesn't have virtualbox guest additions installed
- so need: `vagrant plugin install vagrant-vbguest` on your machine for the synced folders to work