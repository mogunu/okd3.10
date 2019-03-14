# okd3.10
[![N|Solid](https://www.openshift.com/sites/default/files/images/powered-transparent-white.png)](https://okd.io)

### Installing OpenShift Origin 3.10

Step 1. Create the CentOS 7 base image.
Download CentOS minimal ISO image here, create a new VM and install OS in a virtual machine. Hostname set as master and fixed IP address. Once the installation is complete, login (SSH) into the new VM, install updates and common to all nodes packages. This way we ensure that all nodes have the same setup after cloning.

Step 2:

```sh
# sudo yum -y update 
```
Step 3:
```sh
# yum install -y wget git perl net-tools docker-1.13.1 bind-utils iptables-services bridge-utils openssl-devel bash-completion kexec-tools sos psacct python-cryptography python2-pip python-devel python-passlib java-1.8.0-openjdk-headless "@Development Tools"
```
Step 4:
Edit network interface configuration file /etc/sysconfig/network-scripts/ifcfg-eth0 and remove the lines having HWADDR and UUID, also make sure that ONBOOT=”yes”

Step 5:
```sh
# sudo vi /etc/sysconfig/network-scripts/ifcfg-eth0

TYPE="Ethernet"
PROXY_METHOD="none"
BROWSER_ONLY="no"
BOOTPROTO="none"
DEFROUTE="yes"
IPV4_FAILURE_FATAL="no"
IPV6INIT="no"
IPV6_AUTOCONF="yes"
IPV6_DEFROUTE="yes"
IPV6_FAILURE_FATAL="no"
IPV6_ADDR_GEN_MODE="stable-privacy"
NAME="eth0"
DEVICE="eth0"
ONBOOT="yes"
IPADDR="192.168.10.5"
PREFIX="24"
GATEWAY="192.168.10.1"
DNS1="192.168.10.1"
DOMAIN="local"
```
Step 6:
On ansible control server (we can use the same master machine) generate ssh key pair and copy a public key over to master
```sh
[root@master ~]# ssh-copy-id root@master
/usr/bin/ssh-copy-id: INFO: Source of key(s) to be installed: "/root/.ssh/id_rsa.pub"
The authenticity of host 'master (192.168.10.5)' can't be established.
Are you sure you want to continue connecting (yes/no)? yes
/usr/bin/ssh-copy-id: INFO: attempting to log in with the new key(s), to filter out any that are already installed
/usr/bin/ssh-copy-id: INFO: 1 key(s) remain to be installed -- if you are prompted now it is to install the new keys
root@master's password:

Number of key(s) added: 1

Now try logging into the machine, with:   "ssh 'root@master'"
and check to make sure that only the key(s) you wanted were added.

$ ssh root@master

[root@master ~]# logout
Connection to master closed.
```
Step 7: Docker Storage Setup (master).
In virtual machine we need to add one more disk for docker storage (i have just added 100GB) with docker volumes attached as /dev/sdb. We’ll now install and configure docker to use that volume for all local docker storage.

On master server:

$ sudo cat <<EOF > /etc/sysconfig/docker-storage-setup
DEVS=/dev/sdb
VG=docker-vg
EOF

$ sudo docker-storage-setup

## Check results
$ lsblk

$ sudo shutdown -h now
```
Step 8:
On ansible control server test DNS for master server
```sh
[root@master ~]# nslookup master
Server:         192.168.10.1
Address:        192.168.10.1#53

Name:   master.techmogun.local
Address: 192.168.10.5
```
Step 9: Download the OpenShift ansible playbooks.
On ansible control server make sure that the following packages are installed:

“python-passlib” & “httpd-tools”
```sh
yum -y install python-passlib httpd-tools
```
Step 10: 
Clone openshift-ansible:
```sh
$ git clone https://github.com/openshift/openshift-ansible.git
$ cd openshift-ansible
$ git checkout release-3.10
```
Step 11: 
Next we need to create an inventory file where we define target hosts and OpenShift installation strategy. We call this file openshift_inventory and it looks like this:

```sh
[OSEv3:children]
masters
etcd
nodes

[OSEv3:vars]
## Ansible user who can login to all nodes through SSH (e.g. ssh root@master)
ansible_user=root

## Deployment type: "openshift-enterprise" or "origin"
openshift_deployment_type=origin
deployment_type=origin

## Specifies the major version
openshift_release=v3.10
openshift_pkg_version=-3.10.0
openshift_image_tag=v3.10.0
openshift_service_catalog_image_version=v3.10.0
template_service_broker_image_version=v3.10.0
osm_use_cockpit=true
openshift_metrics_install_metrics=True
openshift_logging_install_logging=False

# https://github.com/openshift/origin-metrics/issues/429
openshift_metrics_cassandra_image="docker.io/openshift/origin-metrics-cassandra:v3.11.0"
openshift_metrics_hawkular_metrics_image="docker.io/openshift/origin-metrics-hawkular-metrics:v3.11.0"
openshift_metrics_heapster_image="docker.io/openshift/origin-metrics-heapster:v3.11.0"

# https://github.com/openshift/origin-metrics/issues/429#issuecomment-417124646
openshift_metrics_schema_installer_image="docker.io/alv91/origin-metrics-schema-installer:v3.10.0"

## Service address space,  /16 = 65,534 IPs
openshift_portal_net=172.30.0.0/16

## Pod address space
osm_cluster_network_cidr=10.128.0.0/14

## Subnet Length of each node, 9 = 510 IPs
osm_host_subnet_length=9

## Master API  port
openshift_master_api_port=8443

## Master console port  (e.g. https://console.techmogun.local:443)
openshift_master_console_port=8443

## Clustering method
openshift_master_cluster_method=native

## Hostname used by nodes and other cluster internals
openshift_master_cluster_hostname=master.techmogun.local

## Hostname used by platform users
openshift_master_cluster_public_hostname=master.techmogun.local

## Application wildcard subdomain
openshift_master_default_subdomain=apps.techmogun.local

## identity provider
openshift_master_identity_providers=[{'name': 'htpasswd_auth', 'login': 'true', 'challenge': 'true', 'kind': 'HTPasswdPasswordIdentityProvider'}]

## Users being created in the cluster
openshift_master_htpasswd_users={'admin': '$apr1$5gndYjb1$EypyIDRCBYk75LrsWE5yn', 'user1': '$apr1$7V8/.ewC$yuNOUxZLwQwTeLRiHPKpA'}

## Other vars
containerized=True
os_sdn_network_plugin_name='redhat/openshift-ovs-multitenant'
openshift_disable_check=disk_availability,docker_storage,memory_availability,docker_image_availability

#NFS check bug
openshift_enable_unsupported_configurations=Trueo
#Another Bug 1569476 
skip_sanity_checks=true

openshift_node_kubelet_args="{'eviction-hard': ['memory.available<100Mi'], 'minimum-container-ttl-duration': ['10s'], 'maximum-dead-containers-per-container': ['2'], 'maximum-dead-containers': ['5'], 'pods-per-core': ['10'], 'max-pods': ['25'], 'image-gc-high-threshold': ['80'], 'image-gc-low-threshold': ['60']}"

[OSEv3:vars]

[masters]
master.techmogun.local

[etcd]
master.techmogun.local

[nodes]
master.techmogun.local openshift_schedulable=true ansible_host="{{ lookup('env', '192.168.10.5') }}" openshift_node_group_name="node-config-all-in-one"
```
Step 13: 
Add those values into your inventory file; openshift_master_htpasswd_users & ansible_host ip address.

Step 14:
Install ansible on the master node
```sh
yum -y install epel-release
yum -y install ansible
```

Step 15:
On ansible control host test connectivity to target hosts
```sh
ansible -i openshift_inventory OSEv3 -m ping
master.techmogun.local | SUCCESS => {
"changed": false,
"ping": "pong"
}
```
