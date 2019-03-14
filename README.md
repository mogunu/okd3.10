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
```sh
$ sudo cat <<EOF > /etc/sysconfig/docker-storage-setup
DEVS=/dev/sdb
VG=docker-vg
EOF

$ sudo docker-storage-setup
output will be
[root@master ~]# sudo docker-storage-setup
INFO: Writing zeros to first 4MB of device /dev/sdb
4+0 records in
4+0 records out
4194304 bytes (4.2 MB) copied, 0.231969 s, 18.1 MB/s
INFO: Device node /dev/sdb1 exists.
  Physical volume "/dev/sdb1" successfully created.
  Volume group "docker-vg" successfully created
  Rounding up size to full physical extent 104.00 MiB
  Thin pool volume with chunk size 512.00 KiB can address at most 126.50 TiB of data.
  Logical volume "docker-pool" created.
  Logical volume docker-vg/docker-pool changed.

```
## Check results
```sh
$ lsblk
Output like below
NAME                              MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda                                 8:0    0   50G  0 disk
├─sda1                              8:1    0    1G  0 part /boot
└─sda2                              8:2    0   49G  0 part
  ├─centos-root                   253:0    0 44.2G  0 lvm  /
  └─centos-swap                   253:1    0  4.8G  0 lvm  [SWAP]
sdb                                 8:16   0  100G  0 disk
└─sdb1                              8:17   0  100G  0 part
  ├─docker--vg-docker--pool_tmeta 253:2    0  104M  0 lvm
  │ └─docker--vg-docker--pool     253:4    0 39.8G  0 lvm
  └─docker--vg-docker--pool_tdata 253:3    0 39.8G  0 lvm
    └─docker--vg-docker--pool     253:4    0 39.8G  0 lvm
sr0                                11:0    1 1024M  0 rom

$ sudo shutdown -r now
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
Step 16: We are ready to install OpenShift Origin
Prerequisites will check all dependencies and will start the required services automatically
```sh
ansible-playbook -i openshift_inventory playbooks/prerequisites.yml
```
if it is shown anything in Red please address that before moving to the next step

Step 17: This playbook will install the openshift origin
```sh
ansible-playbook -i openshift_inventory playbooks/deploy_cluster.yml
Output will be
PLAY RECAP ******************************************************************************************************************************
localhost                  : ok=12   changed=0    unreachable=0    failed=0
master.techmogun.local     : ok=829  changed=350  unreachable=0    failed=0


INSTALLER STATUS ************************************************************************************************************************
Initialization              : Complete (0:00:06)
```
Step 18: Post Validation of the installation
```sh
[root@master ~]# oc login -u system:admin -n default
Logged into "https://master.techmogun.local:8443" as "system:admin" using existing credentials.

You have access to the following projects and can switch between them with 'oc project <projectname>':

  * default
    kube-public
    kube-service-catalog
    kube-system
    management-infra
    openshift
    openshift-ansible-service-broker
    openshift-infra
    openshift-logging
    openshift-node
    openshift-sdn
    openshift-template-service-broker
    openshift-web-console

Using project "default".
```
Display the node status:
```sh
[root@master openshift-ansible]# oc get nodes
NAME                     STATUS    ROLES                  AGE       VERSION
master.techmogun.local   Ready     compute,infra,master   12m       v1.10.0+b81c8f8
```

Display all services (note the docker registry network)
```sh
[root@master ~]#  oc get svc
NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                   AGE
docker-registry    ClusterIP   172.30.250.192   <none>        5000/TCP                  9m
kubernetes         ClusterIP   172.30.0.1       <none>        443/TCP,53/UDP,53/TCP     13m
registry-console   ClusterIP   172.30.151.242   <none>        9000/TCP                  9m
router             ClusterIP   172.30.37.136    <none>        80/TCP,443/TCP,1936/TCP   9m
```
Step 18:
Allow insecure access to the docker registry (do it on all nodes)
```sh
vi /etc/sysconfig/docker

INSECURE_REGISTRY='--insecure-registry 172.30.0.0/16'
```
#Restart Docker
```sh
systemctl restart docker
systemctl enable docker
```
Step 19:
add cluster-admin rights to the user
```sh
# oc login -u system:admin -n default

## For a user

[root@master ~]# oc adm policy add-cluster-role-to-user cluster-admin user1
cluster role "cluster-admin" added: "user1"

```
Now we are ready to login OpenShift Origin instance:
https://master.techmogun.local:8443

Password reset for the user
```sh
htpasswd -b /etc/origin/master/htpasswd user1 password
```
