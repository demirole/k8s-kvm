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

### Installing a networking plugin 
The cluster requires a [CNI plugin](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/network-plugins/). Here, we 
will install the [Canal provider from the Calico project](https://docs.projectcalico.org/getting-started/kubernetes/flannel/flannel#installing-with-the-kubernetes-api-datastore-recommended):
```bash
kubectl apply -f https://docs.projectcalico.org/manifests/canal.yaml
```

### Smoke testing cluster
Run your first service: 
```bash
kubectl get nodes
kubectl create deployment nginx --image=nginx
kubectl create service nodeport nginx --tcp=80:80
kubectl describe service nginx
```
You can now reach the nginx services under `http://<IP of master node>:<NodePort>`

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
