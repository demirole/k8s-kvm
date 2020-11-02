# k8s-kvm
Create a Kubernetes cluster with distinct hosts for the control plane and worker nodes on a bare metal
machine running Linux and libvirt + qemu/kvm. 

## Getting started
### Prerequisites
- A bare metal machine with 
  - at least 16GB memory (for 1 master, 2 worker nodes), a32GB recommended)
  - a CPU with at least 4 physical cores
  - with any Linux distro (this guide assumes Ubuntu 18.04)
- Install libvirt, qemu/kvm. E.g., on Ubuntu:
  ```bash
  apt-get install -qy qemu-kvm libvirt-bin virtinst python3 vagrant
  ```
- Start the libvirt daemon. Using either virsh or virt-manager, create a virtual network:
  ```bash
  virsh -c qemu:///system net-create libvirt/default_network.xml
  ```
- Install the `vagrant-libvirt` plugin:
  ```bash
  vagrant plugin install vagrant-libvirt
  ```
- Initialize and install the virtualenv:
  ```bash
  python -m venv venv
  pip install -r requirements.txt
  ```
- Install the kube community module for Ansible
  ```bash
  ansible-galaxy collection install -r requirements.yml
  ```

### Creating the VMs
```bash
vagrant up
```

### Provisioning the VMs
```bash
ansible-playbook -i .vagrant/provisioners/ansible/inventory/vagrant_ansible_inventory playbook.yml
```

### Configure local kubectl
- Install `kubectl` on the local machine
- Configure it
  ```bash
  mkdir -p ~/.kube && vagrant ssh master -c 'sudo cat /etc/kubernetes/admin.conf' > ~/.kube/config
  ```
- Run `kubectl`
  ```bash
  # kubectl cluster-info
  Kubernetes master is running at https://192.168.122.197:6443
  KubeDNS is running at https://192.168.122.197:6443/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy

  To further debug and diagnose cluster problems, use 'kubectl cluster-info dump'.

  # kubectl get nodes                                                                                 
  NAME       STATUS     ROLES    AGE   VERSION
  master     NotReady   master   14m   v1.19.3
  worker-1   NotReady   <none>   13m   v1.19.3
  worker-2   NotReady   <none>   13m   v1.19.3
  worker-3   NotReady   <none>   13m   v1.19.3
  ```
The cluster nodes are marked 'NotReady' because it does not have a network plugin installed yet.

### Configuring the cluster
Run
```bash
ansible-playbook -i .vagrant/provisioners/ansible/inventory/vagrant_ansible_inventory configure-kube-cluster.yml
```

### Smoke testing cluster
The git repository monachus/channel.git by [Adrian Goins](https://adrian.goins.tv/) contains kustomization files 
for deploying a demo application that tests cluster ingress and load balancing. Those files have been made available in this repo.

First, apply the demo project: 
```bash
kubectl apply -k ./cluster-demo
```
 
Then, get the public IP of your cluster and the host name of the demo project:
```bash
kubectl get service ingress-nginx-controller -n ingress-nginx
```
, field `EXTERNAL-IP`, and
```bash
kubectl get ingresses.v1.networking.k8s.io rancher-demo
```
, field 'HOSTS'

Define a DNS lookup in your `/etc/hosts` where the hostname above is resolved to the IP address. Open that 
hostname in your local browser. You should see a webpage that pings one of the three replicas of the demo
project service.
 
Remove the test with
```bash
kubectl delete -k ./cluster-demo
```

### Accessing the Dashboard UI
Create an admin user that can access the Dashboard via a bearer token
```bash
kubectl apply -f kubernetes-manifest/dashboard-adminuser.yaml
kubectl -n kubernetes-dashboard describe secret $(kubectl -n kubernetes-dashboard get secret | grep admin-user | awk '{print $1}')
```

Copy the token. Them, create a proxy
```bash
kubectl proxy
```
and access the [Dashboard UI](http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/#/service?namespace=default``)

## Customisation 
### Change number of nodes
The number of nodes provisioned is defined in the Vagrantfile:
```
NUM_NODES = 3
```
Update the number to get more or less worker nodes. 

### Change VM resources
The number of CPUs and memory allotted for each VM can be change in the section `config.vm.provider`
 of the `Vagrantfile`:
```ruby
config.vm.provider :libvirt do |libvirt|
  libvirt.cpus = 2
  libvirt.memory = 4096
  ...
end
```

## Cleanup
### Destroy the current cluster
Run
```bash
vagrant destroy
```

## Built With
* [Ansible](https://www.ansible.com/) - Simple, agentless IT automation that anyone can use
* [Vagrant](https://www.vagrantup.com/) - Development Environments Made Easy.

## Acknowledgments
* This project is a fork of [talbotfoundry/k8s-kvm](https://github.com/talbotfoundry/k8s-kvm)
* Ingress/LoadBalancer configuration for bare metal setups is beautifully explained by 
[Randal Kamradt Sr](https://medium.com/better-programming/how-to-expose-your-services-with-kubernetes-ingress-7f34eb6c9b5a)
from *Better Programming* 
git@gitlab.com:monachus/channel.git