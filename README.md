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
$ ssh-keygen -t RSA

$ ssh-copy-id root@master
...
root@master's password:

Number of key(s) added: 1

Now try logging into the machine, with: "ssh 'root@master'"
and check to make sure that only the key(s) you wanted were added.

$ ssh root@master

[root@master ~]# logout
Connection to master closed.
```
