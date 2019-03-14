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
