![](http://latincloud.lccv.ufal.br/img/logos/lccv.png)

OpenStack
=========
Cloud computer systems are becoming used by many institutions as a way to keep availiable various kinds of services, like data storage, server hosting, among others. Keeping focus on IaaS ( Infrastructure as a Service), the Openstack stands as an important solution to offer scalability and service availability, supplying a highly flexible infra-structure with fast provisioning.

The OpenStack is a global collaboration of developers and technologists on cloud computing, resulting an ubiquitous and open-source computing platform for both public and private clouds. In practical terms, OpenStack is a set of softwares directed to the configuration and management of a cloud computing environment utilizing existing technologies on Linux environments.


Index
=========

* [1. Infrastructure and Network Settings](#1)
* [2. Preconfiguration Packstack](#2)
  * [2.2 Setting the hostnames](#2.2)
    * [2.2.2 Checking connectivity](#2.2.2)
  * [2.3 Setting the SSH](#2.3)
* [3. Installing and executing Packstack](#3)
  * [3.1 Repositories](#3.1)
  * [3.2 Executing the Packstack Installer](#3.2)
* [4. Post-Installation Configuration](#4)
  * [4.1 Setting the external network](#4.1)
  * [4.2 Creating the external network](#4.2)
  * [4.3 Creating the tenant network (tenant network)](#4.3)
* [5. Instancing a virtual machine](#5)
* [6. Integrate Identity with LDAP](#6)
  * [6.1 Configure Keystone](#6.1)
  * [6.2 Recreating authentication/authorization services](#6.2)
* [7. Important Notes](#7)
* [Referências](#references)

<a name="1"/>
# 1. Infrastructure and Network Settings


Like was said previously, we work on GNU/Linux environment, specifically on CentOS and OpenStack version IceHouse. Thus this guide also works on GNU/Linux Fedora and Red Hat distributions.

We use three RUs SUN FIRE X4170m each one has:
* 2 processors Intel(R) Xeon(R) CPU X5570
* 24GB of RAM
* 128GB of disk
* 2 infiniband interfaces ( high performance interconnection )
* 4 Gigabit Ethernet interfaces
 
The instalation process was done on CentOS 6.5, using the instalation tool [packstack](https://wiki.openstack.org/wiki/Packstack). We consider a multi-node architecture with the Opestack Neutron node which requires the three kinds of nodes:

* **controller**: machine which hosts the managment services (Keystone, Glance, Nova, Horizon...)
* **network**: machine which hosts the network services and is responsible to supply the virtual network and to connect the virtual machines to the external network (neutron).
* **compute**: machine which hosts the virtual machines (hypervisor).


![](https://raw.githubusercontent.com/raphapr/rdo-packstack-installation/master/network.jpg)


In order to install OpenStack Multi-node you are going to need three network interfaces:

* **Management interface** (eth0): Network used to management, not accessible by the external network.
* **Interface for traffic between VMs** (eth1): Network used as internal network for traffic between virtual machines on OpenStack.
* **External interface** (eth2): This network is only connected to the network node in order to supply access to the virtual machines.

<a name="2"/>
# 2. Preconfiguration Packstack

Before instaling OpenStack through packstack, we first prepared the network of the machines

<a name="2.1"/>
## 2.1 Configurating the network 

### Controller node

On node controler, we configurated an interface for the managment (eth0) as follows:

Edit the file  "/etc/sysconfig/network-scripts/ifcfg-eth0" this way:

<pre>
# Internal Network
DEVICE=eth0
TYPE=Ethernet
ONBOOT=yes
NM_CONTROLLED=no
BOOTPROTO=static
IPADDR=192.168.82.11
</pre>

### Network node

On network node, initially we set up two network interfaces.

/etc/sysconfig/network-scripts/ifcfg-eth0

<pre>
DEVICE=eth0
TYPE=Ethernet
ONBOOT=yes
NM_CONTROLLED=no
BOOTPROTO=static
IPADDR=192.168.82.21
NETMASK=255.255.255.0
</pre>

/etc/sysconfig/network-scripts/ifcfg-eth1

<pre>
DEVICE=eth1
TYPE=Ethernet
ONBOOT=yes
NM_CONTROLLED=no
BOOTPROTO=static
IPADDR=192.168.83.21
NETMASK=255.255.255.0
</pre>

### Compute node

On compute node, we set up two network interfaces.

/etc/sysconfig/network-scripts/ifcfg-eth0
<pre>
DEVICE=eth0
TYPE=Ethernet
ONBOOT=yes
NM_CONTROLLED=no
BOOTPROTO=static
IPADDR=192.168.82.31
NETMASK=255.255.255.0
</pre>

/etc/sysconfig/network-scripts/ifcfg-eth1
<pre>
DEVICE=eth1
TYPE=Ethernet
ONBOOT=yes
NM_CONTROLLED=no
BOOTPROTO=static
IPADDR=192.168.83.31
NETMASK=255.255.255.0
</pre>

<a name="2.2"/>
## 2.2 Setting the hostnames

This stage must be done for every node.

After configurating the interfaces, restart the network service.
<pre>
# service network restart
</pre>
Set the machine name:

Edit the line which starts with ''HOSTNAME='' on file ''/etc/sysconfig/network'' as follows:
<pre>
HOSTNAME=controller
</pre>

Edit the file ''/etc/hosts'' as follows:
<pre>
127.0.0.1       localhost
192.168.0.11    controller
192.168.0.21    network
192.168.0.31    compute01
</pre>

<a name="2.2.2"/>
### 2.2.2 Checking connectivity

From every node, ping a web site:

<pre>
# ping -c 4 openstack.org
PING openstack.org (174.143.194.225) 56(84) bytes of data.
64 bytes from 174.143.194.225: icmp_seq=1 ttl=54 time=18.3 ms
64 bytes from 174.143.194.225: icmp_seq=2 ttl=54 time=17.5 ms
64 bytes from 174.143.194.225: icmp_seq=3 ttl=54 time=17.5 ms
64 bytes from 174.143.194.225: icmp_seq=4 ttl=54 time=17.4 ms

--- openstack.org ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3022ms
rtt min/avg/max/mdev = 17.489/17.715/18.346/0.364 ms
</pre>

Ping the network and compute from controller and vice versa.

**Warning: From this point, every configurations are made from controller.**

<a name="2.3"/>
## 2.3 Setting the SSH

Set the SSH access without password through the RSA certificates of controller for the other nodes:

<pre>
# ssh-keygen -t rsa
# ssh-copy-id controller
# ssh-copy-id network
# ssh-copy-id compute01
</pre>

<a name="3"/>
# 3. Instaling and executing Packstack

<a name="3.1"/>
## 3.1 Repositories

Update the current packages on the system:
<pre>
# sudo yum update -y
</pre>

Configure the RDO repositories
<pre>
# sudo yum install -y https://rdo.fedorapeople.org/rdo-release.rpm
</pre>

Install the Packstack installer:
<pre>
# sudo yum install -y openstack-packstack
</pre>

<a name="3.2"/>
## 3.2 Executing the packstack installer

Generate an _answers file_ with all the needed configurations for the installation of OpenStack environment:

<pre>
# packstack --gen-answer-file=packstack-answers-LCCV.txt
</pre>

For this installation, our file  _answer files_ differs from the default by these lines ahead:

<pre>
CONFIG_CINDER_INSTALL=n
CONFIG_NOVA_COMPUTE_HOSTS=192.168.82.31
CONFIG_NEUTRON_SERVER_HOST=192.168.82.21
CONFIG_NEUTRON_LBAAS_HOSTS=192.168.82.11
CONFIG_NEUTRON_OVS_TENANT_NETWORK_TYPE=gre
CONFIG_NEUTRON_OVS_TUNNEL_RANGES=1000:3000
CONFIG_NEUTRON_OVS_TUNNEL_IF=eth1
CONFIG_NEUTRON_ML2_FLAT_NETWORKS=*
</pre>

+ [Complete answer file](https://github.com/raphapr/rdo-packstack-installation/blob/master/packstack-answers-LCCV.txt "Answers file")
+ [Table containing the informations of the configuration keys](https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux_OpenStack_Platform/3/html/Getting_Started_Guide/ch04s03s02.html "Table with info of the configuration keys")

Once the answers file is setted up, start the initialization through the command:

<pre>
# packstack --answer-file=packstack-answers-LCCV.txt
</pre>

<a name="4"/>
# 4. Post-Installation configuration

<a name="4.1"/>
## 4.1 Setting the external network


Assuming the external network is 192.168.84.0/24 and the network interface is **eth2**, edit the following configuration files on **network node**:

/etc/sysconfig/network-scripts/ifcfg-br-ex

<pre>
DEVICE=br-ex
DEVICETYPE=ovs
TYPE=OVSBridge
BOOTPROTO=static
IPADDR=192.168.84.2    # your IP on the external network, may be any.
NETMASK=255.255.255.0  # your netmask
GATEWAY=192.168.84.1   # your gateway
DNS1=192.168.84.25     # your nameserver
ONBOOT=yes
</pre>

/etc/sysconfig/network-scripts/ifcfg-eth2

<pre>
DEVICE=eth2
HWADDR=52:54:00:92:05:AE # MAC address
TYPE=OVSPort
DEVICETYPE=ovs
OVS_BRIDGE=br-ex
ONBOOT=yes
</pre>

Restart the network:

<pre>
# service network restart
</pre>

<a name="4.2"/>
## 4.2 Creating the external network

On **controller node**, Source this file to read in the environment variables:

<pre>
# . /root/keystonerc_admin
</pre>

Inset the following commands:

<pre>
# neutron net-create ext-net --shared --router:external=True
</pre>

<pre>
# neutron subnet-create ext-net --name ext-subnet \
  --allocation-pool start=192.168.84.3,end=192.168.84.254 \
  --disable-dhcp --gateway 192.68.84.1 192.168.84.0/24
</pre>

<a name="4.3"/>
## 4.3 Creating the tenant network (tenant network)

The tenant network is the internal network for VM access, its architecture isolates the network from others. Create it with the following commands:

<pre>
# neutron net-create demo-net 
# neutron net-create demo-net
# neutron subnet-create demo-net --name demo-subnet --dns-nameserver 8.8.8.8 --gateway 10.0.0.1 10.0.0.0/24
</pre>

**Create a router for the internal network and attach it to the external network and internal networks to it.**

<pre>
# neutron router-create demo-router
</pre>

Attach the router to the subnet created:

<pre>
# neutron router-interface-add demo-router demo-subnet
</pre>

Attach the router to the external network by setting it as a gateway:

<pre>
# neutron router-gateway-set demo-router ext-net
</pre>

Make sure the networks were created:

<pre>
# neutron net-list
# neutron subnet-list
</pre>

<a name="5"/>
# 5. Instancing a virtual machine

Register on Nova your public RSA key:

<pre>
# nova keypair-add --pub-key ~/.ssh/id_rsa.pub demo-key
</pre>

Register the CirrOS image of tests on the repositories of Glance:

<pre>
# glance image-create \
  --copy-from http://download.cirros-cloud.net/0.3.1/cirros-0.3.1-x86_64-disk.img \
  --is-public true \
  --container-format bare \
  --disk-format qcow2 \
  --name cirros
</pre>

Add rules to permit ICMP (ping) and SSH access:

<pre>
# neutron security-group-rule-create --protocol icmp default
# neutron security-group-rule-create --protocol tcp \
  --port-range-min 22 --port-range-max 22 default
</pre>

Finaly create the virtual machine:

<pre>
# nova boot --flavor m1.tiny --image cirros --nic net-id=8asc64-d8da-44b6-91cd-4acc87ee3500 --security-group default --key-name demo-key vm_teste
</pre>

See **net-id** of your network via **neutron net-list** command.

Free a floating IP address on the external network.

<pre>
# nova floating-ip-create external
+-------------+-------------+----------+----------+
| Ip          | Instance Id | Fixed Ip | Pool      |
+-------------+-------------+----------+----------+
| 192.168.84.3 | None        | None     | external |
+-------------+-------------+----------+----------+
</pre>

Assign a new instance:

<pre>
# nova add-floating-ip vm_teste 192.168.84.3
</pre>

Check the connectivity via ping/ssh:

<pre>
# ping 192.168.84.3
# ssh cirros@192.168.84.3
</pre>

Default CirroS password: cubswin:)

<a name="6"/>
# 6. Integrate Identity with LDAP

<a name="6.1"/>
## 6.1 Configure Keystone

Set options in the "/etc/keystone/keystone.conf" file. Modify these examples as needed.

<pre>
[identity]
#driver = keystone.identity.backends.sql.Identity
driver = keystone.identity.backends.ldap.Identity
</pre>

Still "keystone.conf", define LDAP server location:

<pre>
[ldap]
url = ldap://localhost
user = dc=Manager,dc=example,dc=org
password = samplepassword
suffix = dc=example,dc=org
use_dumb_member = False
allow_subtree_delete = False
dumb_member=cn=dumb,dc=nonexistent
</pre>

Create the organizational units (OU) in the LDAP directory, and define their corresponding location in the "keystone.conf" file:

<pre>
user_tree_dn = ou=Users,dc=example,dc=org
user_objectclass = inetOrgPerson
user_id_attribute=uid   # usamos uid para fazer a busca no ldap
user_name_attribute=uid #
user_mail_attribute=mail
user_pass_attribute=userPassword
user_attribute_ignore=default_project_id,tenants,enabled
tenant_tree_dn = ou=Groups,dc=example,dc=org
tenant_objectclass = groupOfNames
tenant_id_attribute=cn
tenant_member_attribute=member
tenant_desc_attribute=description
tenant_attribute_ignore=enabled
role_tree_dn = ou=Roles,dc=example,dc=org
role_objectclass = organizationalRole
</pre>

A read-only functions is recommended for LDAP integration. Set on "keystone.conf" file:

<pre>
user_allow_create = False
user_allow_update = False
user_allow_delete = False
 
tenant_allow_create = False
tenant_allow_update = False
tenant_allow_delete = False
 
role_allow_create = False
role_allow_update = False
role_allow_delete = False
</pre>

<a name="6.2"/>
## 6.2 Recreating authentication/authorization services

Keystone has been now configured to use LDAP as the auth backend, in other words, all services will stop 
authenticating until you register them again.

Recreate user, tenant and roles:

<pre>
# export OS_SERVICE_TOKEN=ADMIN_TOKEN
# export OS_SERVICE_ENDPOINT=http://controller:35357/v2.0

# keystone user-create --name=admin --pass=ADMIN_PASS --email=ADMIN_EMAIL
# keystone role-create --name=admin

# keystone tenant-create --name=admin --description="Admin Tenant"
# keystone user-role-add --user=admin --tenant=admin --role=admin

# keystone role-create --name=_member_
# keystone user-role-add --user=admin --role=_member_ --tenant=admin

# keystone tenant-create --name=service --description="Service Tenant"
</pre>

Create a service entry for the Identity Service (keystone):

<pre>
$ keystone service-create --name=keystone --type=identity --description="OpenStack Identity"
</pre>

Specify an API endpoint for the Identity Service:

<pre>
$ keystone endpoint-create --service-id=$(keystone service-list | awk '/ identity / {print $2}') --publicurl=http://controller:5000/v2.0 --internalurl=http://controller:5000/v2.0 --adminurl=http://controller:35357/v2.0
</pre>

This process (create service and specify an API endpoint) is done for each OpenStack service:

<pre>
$ keystone service-list

+----------------------------------+------------+----------------+------------------------------+
|                id                |    name    |      type      |         description          |
+----------------------------------+------------+----------------+------------------------------+
| bb064511680e189nfa9ee0772cb45bec | ceilometer |    metering    |          Telemetry           |
| 60f87d388f142983ddadf467a1dcc78a |   glance   |     image      |   OpenStack Image Service    |
| 44a11ae78add4983b0c8d70713a1ac4f |    heat    | orchestration  |        Orchestration         |
| 82f6f98738734f54b5d89857bc04cc4d |  heat-cfn  | cloudformation | Orchestration CloudFormation |
| 42js4c7f10e9d398ab5573ab330cf6ee |  keystone  |    identity    |  OpenStack Identity Service  |
| 852e2uu2298u26ccbc958baac67216c8 |  neutron   |    network     |     OpenStack Networking     |
| 27f8902f9s124b50be356e656c6870d7 |    nova    |    compute     |      OpenStack Compute       |
| 97719281s5454bf983cvba4031a359aa |  nova_ec2  |      ec2       |   EC2 Compatibility Layer    |
| 93f8ed2826944a1982c7532982chfj22 |   novav3   |   computev3    | Openstack Compute Service v3 |
+----------------------------------+------------+----------------+------------------------------+

</pre>

### Glance

Create a Glance user:

<pre>
# keystone user-create --name=glance --pass=GLANCE_PASS --email=glance@example.com
# keystone user-role-add --user=glance --tenant=service --role=admin
</pre>

Register the service and create the endpoint:

<pre>
# keystone service-create --name=glance --type=image --description="OpenStack Image Service"
# keystone endpoint-create --service-id=$(keystone service-list | awk '/ image / {print $2}') --publicurl=http://controller:9292 --internalurl=http://controller:9292 --adminurl=http://controller:9292
</pre>

### Nova

Create a Nova user:

<pre>
# keystone user-create --name=nova --pass=NOVA_PASS --email=nova@example.com
# keystone user-role-add --user=nova --tenant=service --role=admin
</pre>

Register the service and create the endpoint:

<pre>
# keystone service-create --name=nova --type=compute --description="OpenStack Compute"
# keystone endpoint-create --service-id=$(keystone service-list | awk '/ compute / {print $2}') --publicurl=http://controller:8774/v2/%\(tenant_id\)s --internalurl=http://controller:8774/v2/%\(tenant_id\)s --adminurl=http://controller:8774/v2/%\(tenant_id\)s
</pre>

### Neutron

Create a Neutron user:

<pre>
# keystone user-create --name neutron --pass NEUTRON_PASS --email neutron@example.com
# keystone user-role-add --user neutron --tenant service --role admin
</pre>

Register the service and create the endpoint:

<pre>
# keystone service-create --name neutron --type network --description "OpenStack Networking"
# keystone endpoint-create --service-id $(keystone service-list | awk '/ network / {print $2}') --publicurl http://controller:9696 --adminurl http://controller:9696 --internalurl http://controller:9696
</pre>

### Nova_ec2

Register the service and create the endpoint:

<pre>
# keystone service-create --name nova_ec2 --type ec2 --description "EC2 Compatibility Layer"
# keystone endpoint-create --service-id $(keystone service-list | awk '/ ec2 / {print $2}') --publicurl http://192.168.82.11:8773/services/Cloud --adminurl http://192.168.82.11:8773/services/Admin --internalurl http://192.168.82.11:8773/services/Cloud
</pre>

### Novav3

Register the service and create the endpoint:

<pre>
# keystone service-create --name novav3 --type computev3 --description "Openstack Compute Service v3"
# keystone endpoint-create --service-id $(keystone service-list | awk '/ v3 / {print $2}') --publicurl http://192.168.82.11:8774/v3 --adminurl http://192.168.82.11:8774/v3 --internalurl http://192.168.82.11:8774/v3
</pre>

### Heat

Create a Heat user:

<pre>
# keystone user-create --name=heat --pass=HEAT_PASS --email=heat@example.com
# keystone user-role-add --user=heat --tenant=service --role=admin
</pre>

Register the service and create the endpoint:

<pre>
# keystone service-create --name=heat --type=orchestration --description="Orchestration"
keystone endpoint-create --service-id=$(keystone service-list | awk '/ orchestration / {print $2}') --publicurl=http://controller:8004/v1/%\(tenant_id\)s --internalurl=http://controller:8004/v1/%\(tenant_id\)s --adminurl=http://controller:8004/v1/%\(tenant_id\)s
</pre>

### Heat-cnf

Register the service and create the endpoint:

<pre>
# keystone service-create --name=heat-cfn --type=cloudformation --description="Orchestration CloudFormation"
# keystone endpoint-create --service-id=$(keystone service-list | awk '/ cloudformation / {print $2}') --publicurl=http://controller:8000/v1 --internalurl=http://controller:8000/v1 --adminurl=http://controller:8000/v1
</pre>

### Ceilometer

Create a Ceilometer user:

<pre>
# keystone user-create --name=ceilometer --pass=CEILOMETER_PASS --email=ceilometer@example.com
# keystone user-role-add --user=ceilometer --tenant=service --role=admin
</pre>

Register the service and create the endpoint:

<pre>
# keystone service-create --name=ceilometer --type=metering --description="Telemetry"
# keystone endpoint-create --service-id=$(keystone service-list | awk '/ metering / {print $2}') --publicurl=http://controller:8777 --internalurl=http://controller:8777 --adminurl=http://controller:8777
</pre>

### Restart all services:

<pre>
service openstack-glance-api restart
service openstack-glance-registry restart
service openstack-nova-api start
service openstack-nova-cert start
service openstack-nova-consoleauth restart
service openstack-nova-scheduler restart
service openstack-nova-conductor restart
service openstack-nova-novncproxy restart
service openstack-heat-api restart
service openstack-heat-api-cfn restart
service openstack-heat-engine restart
service openstack-ceilometer-api restart
service openstack-ceilometer-notification restart
service openstack-ceilometer-central restart
service openstack-ceilometer-collector restart
service openstack-ceilometer-alarm-evaluator restart
service openstack-ceilometer-alarm-notifier restart
</pre>

<a name="7"/>
# 7. Important notes

## Is the internet connection too slow on the virtual machines?

On **Network node**, deactivate the *Generic Receive Offloaf* (GRO) on the external interface (eth2) through ethtool:

<pre>
ethtool -K eth2 gro off
</pre>

Add the command on the file /etc/rc.local to make the change to be kept after boot.

## Deleting a network on Neutron

In case you have issues when deleting a network on Neutron, the sequence of commands ahead may be useful:

<pre>
# neutron router-gateway-clear <router-id>
# neutron router-interface-delete <router-id> <tenant-network-id>
# neutron router-delete <router-id>
# neutron net-delete <tenant-network-id>
</pre>

<a name="references"/>
# References

* http://oddbit.com/rdo-hangout-multinode-packstack-slides/
* http://docs.openstack.org/icehouse/install-guide/install/yum/content/
* http://www.monkey-code.com/blog/2013/10/04/building-a-multi-node-openstack-cloud/
* https://access.redhat.com/documentation/en-US/Red_Hat_Enterprise_Linux_OpenStack_Platform/
