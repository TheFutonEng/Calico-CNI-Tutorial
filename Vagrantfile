# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/bionic64"

  config.ssh.insert_key = false

  config.vm.synced_folder ".", "/vagrant", disabled: true

  config.vm.define "vm1" do |vm1|
    vm1.vm.hostname = "vm1.test"
    vm1.vm.network :private_network, ip: "192.168.4.5"
  end
  config.vm.define "vm2" do |vm2|
    vm2.vm.hostname = "vm2.test"
    vm2.vm.network :private_network, ip: "192.168.4.6"
  end
  config.vm.define "etcd-node" do |en1|
    en1.vm.hostname = "etcd-node.test"
    en1.vm.network :private_network, ip: "192.168.4.7"
  end

  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "site.yml"
    ansible.groups = {
      "calico_nodes" => ["vm1", "vm2"],
      "etcd_nodes" => ["etcd-node"],
      "all_nodes:children" => ["calico_nodes", "etcd_nodes"]
    }
  end

#  config.vm.provision "common", type:'ansible' do |ansible|
#    ansible.playbook = "common.yml"
#  end
#
#  config.vm.provision "package-install", type:'ansible' do |ansible|
#    ansible.playbook = "package_install.yml"
#  end
end
 