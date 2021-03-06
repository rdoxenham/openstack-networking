Title: OpenStack Networking Training - Course Manual<br>
Author: Rhys Oxenham - <roxenham@redhat.com><br>
Date: November 2013

#**Course Contents**#

1. **Configuring your host machine for OpenStack**
2. **Deploying Virtual Machines as OpenStack nodes**
3. **Installing OpenStack with Packstack**
4. **Exploring the new environment**
5. **Configuring the Open vSwitch Plugin**
6. **Creating Networks**
7. **Launching Instances**
8. **Creating External Networks**
9. **Managing Floating IP's**
10. **Connecting into Instances**


<!--BREAK-->

#**OpenStack Networking Training Overview**

##**Assumptions**

This manual assumes that you're attending instructor-led training classes and that this material be used as a step-by-step guide on how to successfully complete the lab objectives. Prior knowledge gained from the instructor presentations is highly recommended, however for those wanting to complete this training via their own self-learning, a description at each section is available as well as a copy of the course slides.

The course was written with the assumption that you're learning OpenStack Networking with a Red Hat Enterprise Linux OpenStack Platform base and is written specifically for Red Hat Enterprise Linux (http://www.redhat.com/openstack) although the vast majority of the concepts and instructions will apply to other distributions, including RDO.

The use of a Linux-based hypervisor (ideally using KVM/libvirt) is highly recommended although not essential, the requirements (where necessary) are outlined throughout the course but for deployment on alternative platforms it is assumed that this is already configured by yourself. The course advises that two virtual machines are created, each with their own varying degrees of compute resource requirements so please ensure that enough resource is available. Please note that this course helps you deploy a test-bed for knowledge purposes, it's extremely unlikely that any form of production environment would be deployed in this manor.

By undertaking this course you understand that I take no responsibility for any losses incurred and that you are following the instructions at your own free will. A working knowledge of virtualisation, the Linux command-line, networking, storage and scripting will be highly advantageous for anyone following this guide.

##**What to expect from the course**

Upon completion of the course you should understand the basics of OpenStack and what OpenStack Networking is designed to provide, how Neutron (previously Quantum) and the agents/plugins fit together to enable networking access for instances and how to install/configure them. You should feel comfortable designing OpenStack network architectures and how to position the technology. The course goes into a considerable amount of detail but is far from comprenensive; the target of the course is to provide a solid foundation that can be built upon based on the individuals requirements.

<!--BREAK-->

#**The OpenStack Project**

OpenStack is an open-source Infrastructure-as-a-Service (IaaS) initiative for building and managing large groups of compute instances in an on-demand, massively scale-out cloud computing environment. The OpenStack project, led by the OpenStack Foundation has many goals, most importantly is it's initiative to support interoperability between cloud services and to provide all of the building blocks required to establish a cloud that mimics what a public cloud offers you. The difference being, you get the benefits of being able to stand it up behind a corporate firewall.

The OpenStack project has had a significant impact on the IT industry, its adoption has been very wide spread and has become the basis of the cloud offerings from vendors such as HP, IBM and Dell. Other organisations such as Red Hat, Ubuntu and Rackspace are putting together 'distributions' of OpenStack and offering it to their customers as a supported platform for building a cloud; it's truly seen as the "Linux of the Cloud". The project currently has contributions from developers all over the world, vendors are actively developing plugins and contributing code to ensure that OpenStack can exploit the latest features that their software/hardware exposes.

OpenStack is made up of many individual components in a modular architecture that can be put together to create different types of clouds depending on the requirements of the organisation, e.g. pure-compute or cloud storage. The core components manage access to compute, storage and networking resources.

<!--BREAK-->

#**OpenStack Networking**

OpenStack Networking, like many other components, started off within Nova, as nova-network. It's basic requirement was to provide inbound and outbound networking access to instances running on-top of OpenStack. This encompassed a lot of requirements across most networking layers- an instance needed a network card, it needed access to one or more networks and for that to happen it would need services such as DHCP and DNS, and what about routing? Instances may need to be able to contact other networks, including the Internet. How do we then lock-down access to these instances without having to rely on per-instance firewalling? There were many problems to be solved here, nova-network was the first implementation of a solution, it lasted many major releases and in fact, at the time of writing, it's still an option (in Havana, although deprecated in Icehouse - April 2014). 

Nova-network brought with it the concept of "tenant networks", where individual tenants (or projects) could have their own networking space without having to share with others. The administrators would assign a network range and they could consume resources as they saw fit. These networks were typically considered private, i.e. they existed as networks for instances (essentially virtual machines created within a tenant) to communicate amongst each other. The end-users had limited control over the networks they were assigned but it was a clear distinction between traditional enterprise virtualisation networking and the new-age "networking as a service" model. The implementation of nova-network had three options, the most basic model was called "FlatManager", it simply provided an instance with a mac address but left responsibility for the addressing (e.g. via DHCP) to external services, the extension to this was known as "FlatDHCPManager" which added in a dnsmasq process for providing instances listening on the private network(s) with IP addresses that Nova managed. The problem with these two models was that there was no isolated multi-tenancy; the way that nova-network works is using Linux bridges and despite each tenant having their own network range, they share the same L2 segment (and ARP broadcast domain) and therefore all tenants could "spy" on each other. The alternative option to these was to use "VlanManager" which provided each tenant with VLAN tagged networks therefore isolating the traffic. 

Why do we now hear about Quantum or Neutron as it's now known? Well, the nova-network implementation was very limited, it suited small environments but gave no real control over the networking topology to the end-users, it also didn't make best use of modern day datacenter equipment, i.e. not really an implementation or component of Software Defined Networking (SDN). Neutron (used to be called Quantum in Grizzly) is OpenStack's networking component and is responsible for providing networking to instances running within the OpenStack cloud. In the most basic form, Neutron provides an API to configure and manage a logical network representation, it then relies on agents and plugins to implement these networks in the real world.

One of the main advantages of Neutron is that it (optionally) hands full control of the networking topology over the users of the cloud; they're free to choose IP address ranges, implement virtual routers, virtual load balancers and configure firewalls all within their own isolated virtual network segments. This is, of course, optional - it's possible to configure the networking administratively and allocate networks to the users to consume, or even a combination of both. The networking where we give control over to the users is known as "tenant networks" and when the administrator configures networks for the users based on existing networks these are known as "provider networks"; this guide runs through the two different types and helps you get set-up and familiar with both. 

As mentioned previously, Neutron provides an abstract representation of what networks look like and rely on underlying technologies for the actual implementation; the core component of these technologies is an L2 plugin (and associated agents), in Red Hat Enterprise Linux OpenStack Platform the default to use is Open vSwitch, an open-source software-based virtual switch technology which actually implements the networks that are stored in Neutron. This plugin simply hooks into Neutron and uses agents to make the necessary modifications on the hypervisors. Other plugins are available (https://wiki.openstack.org/wiki/Neutron) although we'll only be discussing Open vSwitch in this guide. Common services such as DHCP, DNS, firewalling and routing are also provided by agents within the Neutron family. 

Note that I explicitly mentioned "routing" there as a major component, when using isolated tenant networks they don't, by default, have any external network access, with Neutron it's possible to configure external network access from tenant networks, e.g. internet access for the instances or external access INTO the tenant networks.
<!--BREAK-->

#**What are we going to be doing?**

We're going to be configuring two virtual machines on-top of a Linux-based (ideally) hypervisor, we'll then use automated mechanisms for deploying a test-bed OpenStack environment and will build up our understanding of OpenStack networking by implementing both tenant networks (with routing) and provider networks.

Note: This course was written for Red Hat Enterprise Linux OpenStack Platform 3.0 (based on Grizzly), hence 'Quantum' is still in use. This guide will be updated for Havana when Red Hat release the updated packages, at which point the commands will start with 'neutron' rather than 'quantum'.
<!--BREAK-->

#**Lab 1: Configuring your host machine for OpenStack**

**Prerequisites:**
* A physical machine installed with either Fedora 19/x86_64 or Red Hat Enterprise Linux 6.4/x86_64
* Alternatively: A hypervisor platform capable of virtualising Linux and providing dedicated network resources

**Tools used:**
* virsh
* yum 

##**Introduction**

This first lab will prepare your local environment for deploying virtual machines that OpenStack will be installed onto; this is considered an "all-in-one" solution, a single physical system where the virtual machines provide the infrastructure. There are a number of tasks that need to be carried out in order to prepare the environment; the OpenStack nodes will need a network to communicate with each other (e.g. a libvirt-NAT interface) and an isolated network which is used to run our tenant networks on (more to come on this later), it will also be extremely beneficial to provide the nodes with access to package repositories via the Internet or repositories available locally, therefore a NAT based network is a great way of establishing network isolation (your hypervisor just becomes the gateway for your OpenStack nodes). The instructions configure a RHEL (or any Linux) based environment to provide this network configuration and make sure we have persistent addresses.

Estimated completion time: 15 minutes


##**Preparing the environment**

In order to be able to use virtual machines we need to make sure that we have the required hardware and software dependencies.

Firstly, check your physical machine supports accelerated virtualisation:

	# egrep '(vmx|svm)' /proc/cpuinfo

Note: You should see either 'vmx' or 'svm' highlighted for you. If it returns nothing, accelerated virtualisation is not present (if may be disabled in the BIOS).

Next, ensure that libvirt and KVM are installed and running.

	# yum install libvirt qemu-kvm virt-manager virt-install -y && chkconfig libvirtd on && service libvirtd start

If you already have an existing virtual machine infrastructure present on your machine, you may want to back-up your configurations and ensure that virtual machines are shutdown to reduce contention for system resources. This guide assumes that you have completed this and you have a more-or-less vanilla libvirt configuration. 

It's important that the 'default' network is defined in a specific way for the guide to be successful:

	# virsh net-info default
	Name            default
	UUID            d81bb3f3-93cc-4269-81c1-f11d6c404fa0
	Active:         yes
	Persistent:     yes
	Autostart:      yes
	Bridge:         virbr0

If this is not present as above (with the exception of a different uuid), it's recommended that you backup your existing default network configuration and recreate it as follows-

	# mkdir -p /root/libvirt-backup/ && mv /var/lib/libvirt/network/default.xml /root/libvirt-backup/
	# virsh net-destroy default && virsh net-undefine default
	# virsh net-define /usr/share/libvirt/networks/default.xml
	# virsh net-edit default

	... change the following section...

  	<ip address='192.168.122.1' netmask='255.255.255.0'>
    		<dhcp>
      			<range start='192.168.122.2' end='192.168.122.254' />
    		</dhcp>
  	</ip>

	...to this...

  	<ip address='192.168.122.1' netmask='255.255.255.0'>
    		<dhcp>
      			<range start='192.168.122.2' end='192.168.122.9' />
	      		<host mac='52:54:00:00:00:01' name='openstack-controller' ip='192.168.122.101' />
      			<host mac='52:54:00:00:00:02' name='openstack-compute' ip='192.168.122.102' />
    		</dhcp>
  	</ip>

	(Use 'i' to edit and when finished press escape and ':wq!')

Then, to save changes, run:

	# virsh net-start default
	# virsh net-autostart default

For ease of connection to your virtual machine instances, it would be prudent to add the machines to your /etc/hosts file-

	# cat >> /etc/hosts <<EOF
	192.168.122.101 openstack-controller
	192.168.122.102 openstack-compute
	EOF

	# virsh net-autostart default
	
Next, create the "isolated network", i.e. the one that we'll use for our tenant networks to run on (will be explained later):

	# cat >> /tmp/isolated.xml << EOF
	  <network>
        <name>isolated</name>
     </network>
	EOF

	# virsh net-define /tmp/isolated.xml
	# virsh net-start isolated
	# virsh net-autostart isolated

Ensure that the bridge is setup correctly on the host:

	# brctl show
	...
	virbr0		8000.5254005a0a54	yes		virbr0-nic
	virbr1		8000.525400bd15a3	yes		virbr1-nic
	# virsh net-info default && virsh net-info isolated
	(See above for correct output)


#**Lab 2: Deploying virtual machines as OpenStack nodes**

**Prerequisites:**
* A physical machine configured with a NAT'd network allocated for hosting virtual machines as well as an isolated vSwitch network

**Tools used:**
* virt command-line tools (e.g. virsh, virt-install, virt-viewer)

##**Introduction**

OpenStack is made up of a number of distinct components, each having their role in making up a cloud. It's certainly possible to have one single machine (either physical or virtual) providing all of the functions, or a complete OpenStack cloud contained within itself. This, however, doesn't provide any form of high availability/resilience and doesn't make efficient use of resources. Therefore, in a typical deployment, multiple machines will be used, each running a set of components that connect to each other via their open API's. When we look at OpenStack there are two main 'roles' for a machine within the cluster, a 'cloud controller' and a 'compute node'. A 'cloud controller' is responsible for orchestration of the cloud, responsibilities include scheduling of instances, running self-service portals, providing rich API's and operating a database store. Whilst a 'compute node' is actually very simple, it's main responsibility is to provide compute resource to the cluster and to accept requests to start instances.

This guide walks you through setting up two machines, one will be the "cloud controller" but will also be our "networking node", i.e. the machine that runs the Neutron server and acts as a gateway to the outside world and the second machine as a dedicated compute host. This section of the guide will quickly provision these machines using Packstack, the OpenStack PoC deployment tool for Red Hat based platforms (https://wiki.openstack.org/wiki/Packstack).

Estimated completion time: 30 minutes


##**Preparing the base content**

Assuming that you have a RHEL 6 x86_64 DVD iso available locally, you'll need to provide it for the installation. Alternatively, if you want to install via the network you can do so by using the '--location http://<path to installation tree>' tag within virt-install.

To save time, we'll install a single virtual machine and just clone it afterwards, that way they're all identical. If you're following this guide with pre-build RHEL (or CentOS/Scientific Linux) images, you can skip the DVD install stages.

##**Without pre-build images**

Only do this if you *don't* have the pre-built images...

	# virt-install --name openstack-controller --ram 2000 --file /var/lib/libvirt/images/openstack-controller.img \
		--cdrom /path/to/dvd.iso --noautoconsole --vnc --file-size 30  --vcpus=2 \
		--os-variant rhel6 --network network:default,mac=52:54:00:00:00:01 --network network:isolated
	# virt-viewer openstack-controller

I would advise that you choose a basic or minimal installation option and don't install any window managers, as these are virtual machines we want to keep as much resource as we can available, plus a graphical view is not required. Partition layouts can be set to default at this stage also, just make sure the time-zone is set correctly and that you provide a root password. When asked for a hostname, I suggest you don't use anything unique, just specify "server" or "node" as we will be cloning.

After the machine has finished installing it will automatically be shut-down, we have to 'sysprep' it to make sure that it's ready to be cloned, this removes any "hardware"-specific elements so that things like networking come up as if they were created individually. In addition, one step ensures that networking comes up at boot time which it won't do by default if it wasn't chosen in the installer.

	# yum install libguestfs-tools -y && virt-sysprep -d openstack-controller
	...

	# virt-edit -d openstack-controller /etc/sysconfig/network-scripts/ifcfg-eth0 -e 's/^ONBOOT=.*/ONBOOT="yes"/'
	
	# virt-clone -o openstack-controller -n openstack-compute -f /var/lib/libvirt/images/openstack-compute.img --mac 52:54:00:00:00:02
	Allocating 'openstack-compute.img'

	Clone 'openstack-compute1' created successfully.
	
##**With pre-built images**

If you have the pre-build RHEL 6 (or CentOS/Scientific Linux) images then you'll just need to import them. Note that the images should ideally have package repositories set-up, this guide assumes that you do:

	# yum install libguestfs-tools -y
	# cp /path/to/rhel6-template.qcow2 /var/lib/libvirt/images/openstack-controller-nonsparse.qcow2
	# virt-sparsify /var/lib/libvirt/images/openstack-controller-nonsparse.qcow2 /var/lib/libvirt/images/openstack-controller.qcow2
	# virt-install --name openstack-controller --ram 2000 --os-variant rhel6 --noautoconsole --vnc \
		--disk /var/lib/libvirt/images/openstack-controller.qcow2,device=disk,bus=virtio,format=qcow2 \
		--network network:default,mac=52:54:00:00:00:01	--network network:isolated --import

Wait for the system to successfully boot up, then clone and prepare the machine...

	# virsh shutdown openstack-controller
	# virt-clone -o openstack-controller -n openstack-compute -f /var/lib/libvirt/images/openstack-compute.img --mac 52:54:00:00:00:02
	# virt-sysprep -d openstack-compute
	# virt-edit -d openstack-compute /etc/sysconfig/network-scripts/ifcfg-eth0 -e 's/^ONBOOT=.*/ONBOOT="yes"/'

Finally, start your virtual machines that'll be used in the next lab and clean-up the temporary file:

	# rm /var/lib/libvirt/images/openstack-controller-nonsparse.qcow2
	# virsh start openstack-controller
	# virsh start openstack-compute
	
#**Lab 3: Installing OpenStack with Packstack**

**Prerequisites:**
* Two running virtual machines with virtual networks attached

**Tools used:**
* ssh
* OpenStack Packstack

##**Introduction**

Packstack is a utility that uses Puppet modules to deploy various parts of OpenStack on multiple pre-installed servers over SSH automatically. Currently only Fedora, Red Hat Enterprise Linux (RHEL) and compatible derivatives are supported.

As this training guide is not about OpenStack in general, we'll use a pre-packaged answer file to build out a known configuration so that the rest of the networking steps work as expected.

Estimated completion time: 35 minutes


##**Installing Packstack**

Firstly, connect into your first machine:

	# ssh root@openstack-controller

This guide assumes that you've already got the required repositories configured. 

	# yum install openstack-packstack facter -y

Download the pre-configured answers file from the git repository and make the necessary adjustments:

	# wget https://raw.github.com/rdoxenham/openstack-networking/master/extras/answers.txt
	# IPADDR=$(facter | grep -m1 ipaddress | awk '{print $3};')
	# sed -i s/changeme/$IPADDR/g answers.txt
	
Make sure that we tell Packstack to configure our second machine as a compute host:	

	# sed -i s/CONFIG_NOVA_COMPUTE_HOSTS=.*/CONFIG_NOVA_COMPUTE_HOSTS=192.168.122.102/g answers.txt
	
Then, execute Packstack with the answer file as a paramater. Note that you'll have to watch the output as it will ask you for root passwords. Before you execute this step it would be prudent to ensure that package repositories are configured correctly on BOTH nodes. Also, run this in screen so that you don't accidentally lose your SSH session:
	
	# screen
	# packstack --answer-file=answers.txt
	
At the time of writing this guide, tunnel networks (which we'll configure in a later chapter) are not currently supported, the packages are available though. On each host:

	# wget http://repos.fedorapeople.org/repos/openstack/openstack-grizzly/epel-6/openvswitch-1.11.0_8ce28d-1.el6ost.x86_64.rpm
	# yum localinstall openvswitch-1.11.0_8ce28d-1.el6ost.x86_64.rpm -y
	
Once completed, reboot the machines as they'll need to be running a specific OpenStack-enabled kernel:

	(return to your host machine)
	# virsh reboot openstack-controller
	# virsh reboot openstack-compute
	
##**So what has been installed for us?**

Packstack has configured the following-

* Keystone - Authentication and authorisation
* Glance - Image repository
* Cinder - Block storage
* Nova - Compute
* Horizon - OpenStack dashboard
* MySQL and Qpid - Supporting infrastructure services
* Created an Admin user and provided a 'source' file in /root (keystonerc_admin)

In addition, it's installed and partly configured Quantum for us. Let's take a look in more depth in the next chapter.


#**Lab 4: Exploring the new environment**

##**Introduction**

In this lab we'll go through the basics of the OpenStack deployment, what Packstack installed for us and what we will need to do further. The answer file that was provided to you as part of this guide is relatively basic, it installs all of the main OpenStack components onto the 'controller' node which is responsible for coordinating the environment and installs the nova-compute service onto the second machine, i.e. the one that will 'do the work' of running instances (virtual machines). 

As Open vSwitch is the default L2 plugin within Red Hat Enterprise Linux OpenStack Platform (or derivatives), Packstack deploys Open vSwitch across all of the nodes that require it, i.e. any node that either runs instances or runs additional network services (e.g. the network controller). As stated in the introductory sections of this guide, Neutron (or Quantum in OpenStack Grizzly) simply stores an abstract/logical representation of defined networks and relies on plugins and agents to implement the topologies "in real life". Open vSwitch agents are responsible for taking commands from Neutron and implementing them on the local machine whilst additional services such as the DHCP or L3 agent provide additional functionality for the networks configured (although are completely optional).

##**Basic Commands**

In root's home directory, Packstack places a 'source' file, i.e. one that stores environment variables that we can use with the OpenStack command-line utilities; it saves us having to pass usernames, passwords and service locations as parameters. For everything we need to do within the environment and every command we need to be authenticated, therefore it's a lot quicker and easier with a source file:

	# ssh root@openstack-controller
	
	# cat /root/keystonerc_admin
	export OS_USERNAME=admin
	export OS_TENANT_NAME=admin
	export OS_PASSWORD=redhat
	export OS_AUTH_URL=http://192.168.122.101:35357/v2.0/
	export PS1='[\u@\h \W(keystone_admin)]\$ '
	
What this tells us is that upon 'usage' of this file it exports a number of environment variables, a username, a password, a tenant (i.e. which project I belong to or want to invoke) and finally the location of Keystone. Note that Keystone really does piece together all of the other components by providing a catalog of services, i.e. the API endpoint or service location of each of the other services, therefore you need only know how to contact Keystone to be able to use OpenStack.

To make use of the source file:

	# source /root/keystonerc_admin
	
You'll now be able to invoke OpenStack commands as the 'admin' user. Note that this user has the in-built 'admin' role, in which the Keystone policy allows to do certain administrative operations, by default any other user created will *not* have this role.

A good start would be to check the services that Packstack has installed for us:

	# keystone service-list
	+----------------------------------+----------+----------+----------------------------+
	|                id                |   name   |   type   |        description         |
	+----------------------------------+----------+----------+----------------------------+
	| 7518cce215304542892c7b9c5431bd0b |  cinder  |  volume  |       Cinder Service       |
	| 48323c5005f9403aa513611ffe038631 |  glance  |  image   |  Openstack Image Service   |
	| 860960f2b6544fc882cfb75973086e91 | keystone | identity | OpenStack Identity Service |
	| e246a2a45d3e446ca6013b02166adbd7 |   nova   | compute  | Openstack Compute Service  |
	| bbd96e91fe1849ae94c6b2ebb28a2cbd | nova_ec2 |   ec2    |        EC2 Service         |
	| 577f2c69da3d4939a9d26f4f66b5eb79 | quantum  | network  | Quantum Networking Service |
	+----------------------------------+----------+----------+----------------------------+
	
We'll use Keystone later to configure a new tenant for us to launch private instances in our own tenant networks later on. For now, let's check out the current network configuration.

We've configured Packstack (in the answer file) to not actually create any networks for us - this guide will explain how to do this manually. As such, the configuration files are minimally altered and the Open vSwitch bridge layout is bare:

	# ovs-vsctl show
	1b881294-4c0c-4ae8-a885-64ed35170a9e
    Bridge br-ex
        Port br-ex
            Interface br-ex
                type: internal
    Bridge br-int
        Port br-int
            Interface br-int
                type: internal
    ovs_version: "1.11.0"
    
What does this tell us? Well, for starters, it's got an entry for two bridges. Firstly, "br-ex" is an external bridge in which we can configure external access for our instances but also inbound access from the outside to our tenant networks via floating IP's. In addition, it's created "br-int", the "integration bridge" in which all LOCAL ports connect into, e.g. virtual machines if this was a compute node, or as this is only a controller the Quantum DHCP and L3 agents would plug in here. The compute node will only have an integration bridge as it's not responsible for routing externally.

We can also view which agents are running across our environment:

	# quantum agent-list
	+--------------------------------------+--------------------+----------------------+-------+----------------+
	| id                                   | agent_type         | host                 | alive | admin_state_up |
	+--------------------------------------+--------------------+----------------------+-------+----------------+
	| 17b3ddf0-7a49-42aa-9763-c1a687cadb68 | Open vSwitch agent | openstack-controller | :-)   | True           |
	| d370c0ab-619c-4d47-865f-27cdc6860d55 | L3 agent           | openstack-controller | :-)   | True           |
	| e0d135bd-176d-447b-be13-c082a60b2241 | DHCP agent         | openstack-controller | :-)   | True           |
	| eef9535a-3b8c-4550-99b3-4f2fccde5929 | Open vSwitch agent | openstack-compute    | :-)   | True           |
	+--------------------------------------+--------------------+----------------------+-------+----------------+

This tells us that whilst the Open vSwitch agent runs on both machines, only the 'openstack-controller' node runs the DHCP and L3 agents, as expected.

#**Lab 5: Configuring the Open vSwitch Plugin**

##**Introduction**

#**Lab 6: Creating Networks**

##**Introduction**

#**Lab 7: Launching Instances**

##**Introduction**

#**Lab 8: Creating External Networks**

##**Introduction**

#**Lab 9: Managing Floating IP's**

##**Introduction**

#**Lab 10: Connecting into Instances**

##**Introduction**

##**Security Groups**

##**Secure Shell Keys**