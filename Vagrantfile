# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
  config.vm.box = "ubuntu/bionic64"

  config.ssh.insert_key = false

  config.vm.synced_folder ".", "/vagrant", disabled: true

  config.vm.define "vm1" do |app|
    app.vm.hostname = "vm1"
  end
  config.vm.define "vm2" do |app|
    app.vm.hostname = "vm2"
  end
  config.vm.define "etcd-node" do |app|
    app.vm.hostname = "etcd-node"
  end

  config.vm.provision "ansible" do |ansible|
    ansible.playbook = "site.yml"
    ansible.groups ={}

#  config.vm.provision "common", type:'ansible' do |ansible|
#    ansible.playbook = "common.yml"
#  end
#
#  config.vm.provision "package-install", type:'ansible' do |ansible|
#    ansible.playbook = "package_install.yml"
#  end
end
 