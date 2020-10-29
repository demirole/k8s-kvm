ENV['VAGRANT_DEFAULT_PROVIDER'] = 'libvirt'
IMAGE_NAME = "generic/ubuntu1804"

NETWORK_PREFIX="192.168.122"
NUM_NODES = 3

def ip_from_num(network_prefix, counter)
    "#{network_prefix}.#{100+counter}"
end

def get_nodes(network_prefix, nof_nodes)
    nodes = Array[]
    (1..nof_nodes).each do |i|
        nodes << {
            :name => "worker-#{i}",
            :ip => ip_from_num(network_prefix, i)
        }
    end
    nodes
end

master = {
    :name => "master",
    :ip => ip_from_num(NETWORK_PREFIX, 0)
}

nodes = get_nodes(NETWORK_PREFIX, NUM_NODES)
node_names = nodes.collect{|n| n[:name]}

Vagrant.configure("2") do |config|
    config.ssh.insert_key = false

    config.vm.provider :libvirt do |libvirt|
      libvirt.cpus = 2
      libvirt.memory = 4096
      libvirt.driver = "kvm"
      libvirt.management_network_name = "default"
      libvirt.management_network_address = "#{NETWORK_PREFIX}.0/24"
    end
      
    config.vm.define master[:name] do |m|
        m.vm.box = IMAGE_NAME
        m.vm.hostname = master[:name]
    end

    nodes.each do |node|
        config.vm.define node[:name] do |n|
            n.vm.box = IMAGE_NAME
            n.vm.hostname = node[:name]
        end
    end

    config.vm.provision "ansible" do |ansible|
        ansible.playbook = "create_auto_inventory.yml"
        ansible.groups = {
            "kube-master" => master[:name],
            "kube-nodes" => node_names
        }
    end
end
