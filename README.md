# Introduction
The inspiration for this repo came from [this Medium post from Vikram Fugro](https://medium.com/@vikram.fugro/project-calico-the-cni-way-659d057566ce) ([@cotigao](https://github.com/cotigao) on Github) on the subject of investigating Project Calico using Ubuntu servers.  It is highly recommended that this be read first before proceeding.  This repo merely provides a means using Vagrant and Ansible to quickly get to the point where one can learn about Calico, CNI and the interaction between them.  It is not meant to be a deep dive into Vagrant or Ansible but some pointers may be highlighted if they're applicable to the flow of the project.

# Required Tools 
The tools in the below table are what are required to be installed on a laptop/computer in order to be able to use this repo.  Note, future editions of these tools may work just fine, these are the versions that were tested.

| Tool      | Tested Version | Install Instructions  |
| --------- |:-------:|:---------------------:|
| Vagrant   | 2.2.10  | [link](https://www.vagrantup.com/docs/installation) |
| Ansible   | 2.9     | [link](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html) |
| Virtualbox| 6.1.12  | [link](https://www.virtualbox.org/manual/ch02.html) |
| Sufficient resource to run three VMs | N/A | N/A |
| An Internet connetion | N/A | N/A |

TODO:Test with configurations to limit the amount of RAM Vagrant provides to each system to be 256MB.

For those unfamiliar with Vagrant, it is a product from Hashhicorp which allows for VMs or containers to be spun up and subsequently provisioned for the purposes of creating a lightweight and portable development environment.  The reason for listing Virtualbox above is that Vagrant by default uses Vitualbox as its hypervisor for spinning up resources.  Other tools can be used for this purpose but this repo was developed for Virtualbox.

# Topology
![alt test](https://github.com/TheFutonEng/Calico-CNI-Tutorial/raw/calico-cni-tutorial_topology.png "Calico CNI Tutorial Topology") 

# Setup
Once all of the above tools are installed on the target test system, clone the repo to the target test system:

```bash
$ git clone git@github.com:TheFutonEng/Calico-CNI-Tutorial.git
```

After that, run the following to deploy the three VMs:

```bash
$ cd Calico-CNI-Tutorial
$ vagrant up
```

This command may take a few minutes to run but it does the following:

* Builds the three VMs defined in the Vagrantfile and detailed in the previous topology section  
* Stands up an etcd docker container on the etcd node 
* Stands up calico docker containers on the calico nodes which point to the etcd node for their datastore
* Installs calicoctl on all three VMs  
* Downloads and installs the calico and calico-ipam CNI reference plugins on the calico nodes
* Downloads and installs Golang on the calico nodes (role to perform this copied from fubarhouse [here](https://github.com/fubarhouse/ansible-role-golang))
* Downloads and installs the CNI network plugins on the calico nodes
* Downloads and install the cnitool on the calico nodes (install locaiton is /home/vagrant/go/bin/cnitool)

# Setup Verification

Run the this section to confirm that the setup has worked.

```
$ vagrant ssh etcd-node
$ sudo calicoctl get nodes
```

The first command uses an SSH key generated by vagrant to log into the etcd-node.  The second command checks what nodes have registered with etcd, the output should be the two calico nodes.  Note, it may take a minute or two for calico nodes to register with the etcd node after the ```vagrant up``` command completes.

```
NAME      
calico1   
calico2 
```

Now access one of the calico nodes and confirm that it has discovered the other node:

```
$ vagrant ssh calico1
$ sudo calicoctl node status
```

The output from the command should be the data regarding the established BGP peer with calico2:

```
Calico process is running.

IPv4 BGP status
+--------------+-------------------+-------+----------+-------------+
| PEER ADDRESS |     PEER TYPE     | STATE |  SINCE   |    INFO     |
+--------------+-------------------+-------+----------+-------------+
| 192.168.4.6  | node-to-node mesh | up    | 23:26:50 | Established |
+--------------+-------------------+-------+----------+-------------+

IPv6 BGP status
No IPv6 peers found.
```

# Calico and the CNI - Exploration

From this point, feel free to explore however you wish.  This repo will continue to step through some changes that can be done once the setup is up and running successfully.  Automation has been written to perform each of the steps below and the instructions on using that automation will be supplied first.  Manual instructions for each step will be supplied thereafter if you would prefer to step through each change on your own for learning purposes.

## IPPool

An IP Pool is a Calico resource type which represents an IP range from which Calico can assign individual addresses to endpoints.  In order to create a pool, run the following command from your localhost machine in the root directory of the repo (Calico-CNI-Tutorial/)

### Automated IPPool Deployment

```bash
$ ansible-playbook -i .vagrant/provisioners/ansible/inventory/vagrant_ansible_inventory ippool.yml 
```

### Manual IPPool Deployment

This command will deploy the ippool defined in: 

```Calico-CNI-Tutorial/roles/ippool/templates/ippool.yml.j2```

The equivalent command that can be run from the CLI of the etcd-node is below for reference:

```bash 
$ calicoctl create -f ippool.yml
```

Note, the above command assumes that ippool.yml exists on the etcd-node.  A sample YAML file is below for reference (copied from the Medium post referenced in the introduction):

```yaml
apiVersion: projectcalico.org/v3
kind: IPPool
metadata:
  name: my.ippool-1
spec:
  cidr: 10.1.0.0/16
  ipipMode: Never
  natOutgoing: true
  disabled: false
  blockSize: 26
```

## Calico Configuration

This section covers getting the network plumbing in place for the calico containers.  Note, Calico only supports CNI spec versions 0.1.0, 0.2.0, 0.3.0 and 0.3.1 so if you've strayed from the repo (and that's ok!) be sure that your CNI configuration file leverages one of these versions.

Make a mental note here on what interfaces are on the Calico nodes when running the ```ip addr``` command.  This will be referenced shortly.

```
vagrant@calico1:~$ ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 02:df:14:00:a4:ed brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 80134sec preferred_lft 80134sec
    inet6 fe80::df:14ff:fe00:a4ed/64 scope link 
       valid_lft forever preferred_lft forever
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:ad:b1:52 brd ff:ff:ff:ff:ff:ff
    inet 192.168.4.5/24 brd 192.168.4.255 scope global enp0s8
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fead:b152/64 scope link 
       valid_lft forever preferred_lft forever
4: docker0: <NO-CARRIER,BROADCAST,MULTICAST,UP> mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:f5:98:0a:06 brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
vagrant@calico1:~$
```

### Automated Calico Networking Deployment

The below command run from your laptop/desktop will plumb an interface into the docker containers running on the Calico nodes:

```
$ ansible-playbook -i .vagrant/provisioners/ansible/inventory/vagrant_ansible_inventory calico_networking.yml 
```

### Manual Calico Networking Deployement

To achieve this result manually, run the following steps on each Calico node:

1. Get the container id for the Calico container

```
$ docker inspect calico-release-v3.14 | jq .[0].Id | tr -d '"'
```

Example Output:
```
vagrant@calico1:~$ sudo docker inspect calico-release-v3.14 | jq .[0].Id | tr -d '"'
d4c5666d4fe74a5f298ffbc0ec31e1eb57726cef14f42211d44bd9d2fcabbe62
vagrant@calico1:~$
```

2. Get the SandboxKey for the Calico container

```
$ sudo docker inspect calico-release-v3.14 | jq .[0].NetworkSettings.SandboxKey | tr -d '"'
```

Example Output:
```
vagrant@calico1:~$ sudo docker inspect calico-release-v3.14 | jq .[0].NetworkSettings.SandboxKey | tr -d '"'
/var/run/docker/netns/default
vagrant@calico1:~$
```

3. Call the cnitool command to add an interface to the container

```
$ sudo CNI_PATH=/home/vagrant/go/bin NETCONFPATH=/etc/cni/net.d CNI_CONTAINERID=<<<output from step 1>>  /home/vagrant/go/bin/cnitool add test-calico <<output from step 2>
```

Example Output:
```
vagrant@calico1:~$ sudo CNI_PATH=/home/vagrant/go/bin NETCONFPATH=/etc/cni/net.d CNI_CONTAINERID=d4c5666d4fe74a5f298ffbc0ec31e1eb57726cef14f42211d44bd9d2fcabbe62  /home/vagrant/go/bin/cnitool add test-calico /var/run/docker/netns/default
{
    "cniVersion": "0.3.1",
    "interfaces": [
        {
            "name": "calicnitool-573"
        }
    ],
    "ips": [
        {
            "version": "4",
            "address": "10.1.127.192/32"
        }
    ],
    "dns": {}
}vagrant@calico1:~$
```

### Explore What Just Happened

Note the that address displayed in the ```ips``` section in the above output was pulled out of the ippool that was created in the previous section.  Running the ```ip addr``` command directly on one of the Calico nodes now will show a new interface:

<pre>
vagrant@calico1:~$ ip addr
1: lo: &lt;LOOPBACK,UP,LOWER_UP&gt; mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: enp0s3: &lt;BROADCAST,MULTICAST,UP,LOWER_UP&gt; mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 02:df:14:00:a4:ed brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 46827sec preferred_lft 46827sec
    inet6 fe80::df:14ff:fe00:a4ed/64 scope link 
       valid_lft forever preferred_lft forever
3: enp0s8: &lt;BROADCAST,MULTICAST,UP,LOWER_UP&gt; mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 08:00:27:3d:e7:68 brd ff:ff:ff:ff:ff:ff
    inet 192.168.4.5/24 brd 192.168.4.255 scope global enp0s8
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe3d:e768/64 scope link 
       valid_lft forever preferred_lft forever
4: docker0: &lt;NO-CARRIER,BROADCAST,MULTICAST,UP&gt; mtu 1500 qdisc noqueue state DOWN group default 
    link/ether 02:42:f0:1e:b2:9c brd ff:ff:ff:ff:ff:ff
    inet 172.17.0.1/16 brd 172.17.255.255 scope global docker0
       valid_lft forever preferred_lft forever
<b>7: calicnitool-573@eth0: &lt;BROADCAST,MULTICAST,UP,LOWER_UP&gt; mtu 1500 qdisc noqueue state UP group default 
    link/ether ee:ee:ee:ee:ee:ee brd ff:ff:ff:ff:ff:ff
    inet6 fe80::ecee:eeff:feee:eeee/64 scope link 
       valid_lft forever preferred_lft forever
8: eth0@calicnitool-573: &lt;BROADCAST,MULTICAST,UP,LOWER_UP&gt; mtu 1500 qdisc noqueue state UP group default 
    link/ether 9a:24:70:ab:f8:eb brd ff:ff:ff:ff:ff:ff
    inet 10.1.127.192/32 scope global eth0
       valid_lft forever preferred_lft forever
    inet6 fe80::9824:70ff:feab:f8eb/64 scope link 
       valid_lft forever preferred_lft forever</b>
vagrant@calico1:~$ 
</pre>

Entering into the container shows a similar output:

```
$ sudo docker exec -it calico-release-v3.14 /bin/bash
```

From there, running the ```ip addr``` command on both containers produces similar output as running the ```ip addr``` command on the VM directly.  To continue with this example, the IP addresses handed out for each containers in this particular run are noted below:

* calico1 - 10.1.127.192
* calico2 - 10.1.86.64

The commmand that shows the path best here is shown below with it's output:

```
[root@calico1 /]# ip route show | grep 10.1.86.64
10.1.86.64/26 via 192.168.4.6 dev enp0s8 proto bird 
[root@calico1 /]# 
```

This is run from within the container on calico1 and is looking for the IP address for calico2 in it's output.  There are a couple of things to notice here:

1. The route is actually for a /26 and not a host route

#### Explanation

The reason for this has to do with the blockSize field set in the ippool that was created.  Straight from the [Calico website](https://docs.projectcalico.org/reference/resources/ippool):

> "This allows addresses to be allocated in groups to workloads running on the same host. By grouping addresses, fewer routes need to be exchanged between hosts and to other BGP peers. If a host allocates all of the addresses in a block then it will be allocated an additional block. If there are no more blocks available then the host can take addresses from blocks allocated to other hosts. Specific routes are added for the borrowed addresses which has an impact on route table size."

What happens if another container is spun up and plumbed on either Calico node?  Give it a shot.

2. The next hop is the underlay address for calico2 (192.168.4.6) out of underlay interface for calico1 (enp0s8)

#### Explanation

This is at the very heart of how Calico behaves and is again best explained by [Calico's documentation](https://docs.projectcalico.org/reference/architecture/data-path).  In short, Calico helps containers route to where the workload is:

> "In the Calico approach, IP packets to or from a workload are routed and firewalled by the Linux routing table and iptables infrastructure on the workload’s host. For a workload that is sending packets, Calico ensures that the host is always returned as the next hop MAC address regardless of whatever routing the workload itself might configure. For packets addressed to a workload, the last IP hop is that from the destination workload’s host to the workload itself."

3. The protocol which programmed the route is bird

#### Explanation

This observation is the logical conclusion of the first two points.  The reason it's "bird" in this repo is because that's what was set as the ```calico_backend``` variable in the calico-install Ansible role (exact location within this repo where that variable is defined is Calico-CNI-Tutorial/roles/calico-install/defaults/main.yml).  Bird uses BGP in order to exchange routes so this route was programmed by an actual dynamic routing protocol.  For additional 

# Additional Resources

* [Kubernetes and the CNI](https://www.caseyc.net/cni-talk-kubecon-18.pdf)
* [Linux VETH man page](https://man7.org/linux/man-pages/man4/veth.4.html)
* [The Calico Data Path](https://docs.projectcalico.org/reference/architecture/data-path)
* [Configure the Calico CNI Plugins](https://docs.projectcalico.org/reference/cni-plugin/configuration)