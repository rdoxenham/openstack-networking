Title: OpenStack Networking Training - Course Manual<br>
Author: Rhys Oxenham <roxenham@redhat.com><br>
Date: November 2013

#**Course Contents**#

1. **Configuring your host machine for OpenStack**
2. **Deploying virtual-machine instances as base infrastructure**


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
	# virsh net-start default

Ensure that the bridge is setup correctly on the host:

	# brctl show
	...
	virbr0		8000.5254005a0a54	yes		virbr0-nic

	# virsh net-info default
	(See above for correct output)


