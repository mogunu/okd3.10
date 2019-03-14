# okd3.10
[![N|Solid](https://techm.000webhostapp.com/wp-content/uploads/2017/06/cropped-logo.png](https://techmogun.com)
Installing OpenShift Origin

Step 1. Create the CentOS 7 base image.
Download CentOS minimal ISO image here, create a new VM and install OS in a virtual machine. Hostname set as master and fixed IP address. Once the installation is complete, login (SSH) into the new VM, install updates and common to all nodes packages. This way we ensure that all nodes have the same setup after cloning.

Step 2:
sudo yum -y update 

Step 3:
yum install -y wget git perl net-tools docker-1.13.1 bind-utils iptables-services bridge-utils openssl-devel bash-completion kexec-tools sos psacct python-cryptography python2-pip python-devel python-passlib java-1.8.0-openjdk-headless "@Development Tools"

