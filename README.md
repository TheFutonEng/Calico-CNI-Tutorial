# Calico-CNI-Tutorial

# Introduction
The inspiration for this repo came from [this Medium post from Vikram Fugro](https://medium.com/@vikram.fugro/project-calico-the-cni-way-659d057566ce) on the subject of investigating Project Calico using Ubuntu servers.  This repo merely provides a means using Vagrant and Ansible to quickly get to the point where one can learn about Calico, CNI and the interaction between them.  It is not meant to be a deep dive into Vagrant or Ansible but some pointers will be highlighted if they're applicable to the flow of the project.

# Required Tools 
The tools in the below table (hopefully a table) are what are required to be installed on a laptop/computer in order to be able to use this repo.  Note, future editions of these tools may work just fine, these are the versions that were tested.

See about inserting a table for this with links to the project site and installation instructions for each.

| Tool      | Tested Version | Install Instructions  |
| --------- |:-------:|:---------------------:|
| Vagrant   | 2.2.10  | [link](https://www.vagrantup.com/docs/installation) |
| Ansible   | 2.9     | [link](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) |
| Virtualbox| 6.1.12  | [link](https://www.virtualbox.org/manual/ch02.html) |
| Sufficient resource to run three VMs | N/A | N/A |
| An Internet connetion | N/A | N/A |

TODO:Test with configurations to limit the amount of RAM Vagrant provides to each system to be 256MB.

For those unfamiliar with Vagrant, it is a product from Hashhicorp which allows for VMs or containers to be spun up and subsequently provisioned.  The reason for listing Virtualbox above is that Vagrant by default (fact check this) uses Vitualbox as its hypervisor for spinning up resources.  

# Topology
TODO: Create a diagram of the VMs and docker containers that get launched

# Setup
Once all of the above tools are installed on the target test system, clone the repo to the target test system:

```bash
git clone git@github.com:TheFutonEng/Calico-CNI-Tutorial.git
```

After that, run the following to deploy the three VMs:

```bash
cd Calico-CNI-Tutorial
vagrant up
```

This command may take a few minutes to run.  It builds the three VMs defined in the Vagrantfile and detailed in the previous topology section.  Additionally it runs the common.yml ansible playbook which gets called in the Vagrantfile as a provisioner.  

The reason site.yml (or the "do everything" playbook) wasn't called as part of the provisioner is that Vagrant runs the tasks in serial fashion against one machine at a time.  In previous research, it seemed like the Etcd node needs to be online before the Calico nodes.  Vagrant does have a means of running Ansible tasks in parallel and having multiple provisioners (see [this page](https://www.vagrantup.com/docs/provisioning/ansible.html) for details).  But running tasks in parallel is something that Ansible can natively do so no need to reinvent the wheel there.

After the three VMs hasve been provisioned and common.yml has been run against them by the vagrant provisioner, run site.yml

```bash
ansible-playbook -i inventory site.yml
```

This will configure first the etcd node and then the calico nodes.
