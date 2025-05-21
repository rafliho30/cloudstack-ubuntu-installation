---
title: Cloudstack Installation

---

# Apache CloudStack: Private Cloud Installation

**Members**:
1. Rafli Adhitia
1. Darren Adam Dewantoro
1. Darmawan Hanif
1. Kevin Raihan

## Project Overview
Apache CloudStack is an open-source Infrastructure-as-a-Service (IaaS) platform that enables the creation and management of scalable public or private cloud environments. It orchestrates pools of compute, storage, and networking resources, supporting multiple hypervisors including KVM, VMware vSphere, Hyper-V, and XenServer.

### Features

* **Multi-Hypervisor Support**: Integrates with various hypervisors, allowing flexibility in deployment.
* **Scalability**: Manages thousands of servers across geographically distributed data centers.
* **Automated Configuration**: Handles network and storage settings for virtual machines automatically.
* **User Interfaces**: Provides both a customizable web-based GUI and a REST-like API for cloud management.
* **High Availability**: Ensures continuous operation of virtual machines even during management server outages.

### System Architecture

#### 1. **Zone (Data Center Level)**

* Represents a physical data center.
* Contains pods, primary storage, and secondary storage.
* Acts as the highest-level organizational unit.

#### 2. **Pod (Rack Level)**

* Represents a group of servers in the same network subnet.
* Located within a zone.
* Contains one or more clusters and a Layer-2 switch.

#### 3. **Cluster (Hypervisor + Storage Group)**

* A group of hosts with the same hypervisor type.
* Shares the same primary storage.
* Provides VM HA (High Availability) capabilities.

#### 4. **Host (Physical Server)**

* A server running a hypervisor (e.g., KVM, VMware, XenServer).
* Provides compute resources to virtual machines.

#### 5. **Storage**

* **Primary Storage**: Attached to clusters, used for storing VM volumes and templates.
* **Secondary Storage**: Shared across a zone, used for storing ISO images, VM templates, and snapshots.

#### 6. **Networking**

* **Basic Networking**: Flat network without VLANs; all VMs share a single network.
* **Advanced Networking**: Supports VLANs, virtual routers, and isolated networks per account.

This project involves setting up a private cloud environment using Apache CloudStack. It covers the installation and configuration of key components, including the Management Server, KVM hypervisor, NFS primary and secondary storage, and a virtual router. The goal is to create a scalable, self-managed cloud infrastructure for resource provisioning and management.

### Requirement

- CPU : Intel Core i5 gen 8
- RAM : 24 GB
- Storage : 250GB
- Network : Ethernet 100GB/s
- Operating System : Ubuntu Server 22.04




## Pre-installation
### Setting up SSH
First we install net-tools and ssh to be able to config our server remotely
```
sudo apt update
sudo apt install net-tools openntpd openssh-server sudo vim tar -y
sed -i '/#PermitRootLogin prohibit-password/a PermitRootLogin yes' /etc/ssh/sshd_config
sudo vim /etc/ssh/sshd_config
```
Find the line 'PermitRootLogin' make sure it set to 'yes'. Make sure that its uncommented
```
sudo systemctl enable ssh
sudo systemctl start ssh
sudo ufw allow ssh
```
Next we check our ip address using **ifconfig**
```
ifconfig
```
We get the IP address 192.168.1.88
To connect to our server we can use the following command on our device : 
```
ssh kelompok18@192.168.1.88
```

### Installing hardware resource monitoring tools
```
sudo apt install htop lynx duf -y
sudo apt install bridge-utils
sudo apt-get install intel-microcode -y
```

### Configure network
```
cd /etc/netplan
sudo nano ./0*.yaml
```
The opened file will look like this:
![image](https://hackmd.io/_uploads/B1V3gqZbxx.png)

### Configure LVM
```
#not required unless logical volume is used
lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
resize2fs /dev/ubuntu-vg/ubuntu-lv
```

## Cloudstack Installation

apt-get install openntpd openssh-server sudo vim tar -y

```
nano /etc/ssh/sshd_config
    PermitRootLogin yes
    #PermitRootLogin prohibit-password (comment bagian ini)
systemctl restart sshd.service
```

![image](https://hackmd.io/_uploads/r1njZ9-Wge.png)

```
sudo -i
mkdir -p /etc/apt/keyrings 
wget -O- http://packages.shapeblue.com/release.asc | gpg --dearmor | sudo tee /etc/apt/keyrings/cloudstack.gpg > /dev/null
echo deb [signed-by=/etc/apt/keyrings/cloudstack.gpg] http://packages.shapeblue.com/cloudstack/upstream/debian/4.18 / > /etc/apt/sources.list.d/cloudstack.list
```

![image](https://hackmd.io/_uploads/H1jpfcWbeg.png)
```
apt-get update -y
apt-get install cloudstack-management mysql-server
```

```
nano /etc/mysql/mysql.conf.d/mysqld.cnf
```
![image](https://hackmd.io/_uploads/rkcmXcZZxl.png)

```
cloudstack-setup-databases cloud:cloud@localhost --deploy-as=root:kelompok18 -i 192.168.106.250
```
Configure Primary & Secondary Storage
```
apt-get install nfs-kernel-server quota
echo "/export  *(rw,async,no_root_squash,no_subtree_check)" > /etc/exports
mkdir -p /export/primary /export/secondary
exportfs -a
```
Configure NFS Server
```
sed -i -e 's/^RPCMOUNTDOPTS="--manage-gids"$/RPCMOUNTDOPTS="-p 892 --manage-gids"/g' /etc/default/nfs-kernel-server
sed -i -e 's/^STATDOPTS=$/STATDOPTS="--port 662 --outgoing-port 2020"/g' /etc/default/nfs-common
echo "NEED_STATD=yes" >> /etc/default/nfs-common
sed -i -e 's/^RPCRQUOTADOPTS=$/RPCRQUOTADOPTS="-p 875"/g' /etc/default/quota
service nfs-kernel-server restart
```
![image](https://hackmd.io/_uploads/rJLNPhbWee.png)
![image](https://hackmd.io/_uploads/ByRaD3Zbll.png)


## Configure Cloudstack Host with KVM Hypervisor
### Install KVM and Cloudstack Agent
```
apt-get install qemu-kvm cloudstack-agent -y
```


```
sed -i -e 's/\#vnc_listen.*$/vnc_listen = "0.0.0.0"/g' /etc/libvirt/qemu.conf

# On Ubuntu 22.04, add LIBVIRTD_ARGS="--listen" to /etc/default/libvirtd instead.

sed -i.bak 's/^\(LIBVIRTD_ARGS=\).*/\1"--listen"/' /etc/default/libvirtd
```

### Restart Libvirtd
```
systemctl mask libvirtd.socket libvirtd-ro.socket libvirtd-admin.socket libvirtd-tls.socket libvirtd-tcp.socket
systemctl restart libvirtd
```
### Configuration to Support Docker and Other Services
```
echo "net.bridge.bridge-nf-call-arptables = 0" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables = 0" >> /etc/sysctl.conf
sysctl -p
```

### Generate Unique Host ID
```
apt-get install uuid -y
UUID=$(uuid)
echo host_uuid = \"$UUID\" >> /etc/libvirt/libvirtd.conf
systemctl restart libvirtd
```

### Configure Iptables Firewall and Make it persistent
```
NETWORK=192.168.1.50/24
iptables -A INPUT -s $NETWORK -m state --state NEW -p udp --dport 111 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 111 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 2049 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 32803 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p udp --dport 32769 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 892 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 875 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 662 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 8250 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 8080 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 8443 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 9090 -j ACCEPT
iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 16514 -j ACCEPT
#iptables -A INPUT -s $NETWORK -m state --state NEW -p udp --dport 3128 -j ACCEPT
#iptables -A INPUT -s $NETWORK -m state --state NEW -p tcp --dport 3128 -j ACCEPT

apt-get install iptables-persistent
#just answer yes yes
```

### Disable apparmour on libvirtd
```
ln -s /etc/apparmor.d/usr.sbin.libvirtd /etc/apparmor.d/disable/
ln -s /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper /etc/apparmor.d/disable/
apparmor_parser -R /etc/apparmor.d/usr.sbin.libvirtd
apparmor_parser -R /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper
```

### Launch management Server
```
cloudstack-setup-management
systemctl status cloudstack-management
tail -f /var/log/cloudstack/management/management-server.log #if you want to troubleshoot 
#wait until all services (components) running successfully
```

### Open Web Browser
```
http://<YOUR_IP_ADDRESS>:8080
```
![image](https://hackmd.io/_uploads/BkHCFqbZgx.png)

Login dengan default username
```
username : admin
password : password
```
![image](https://hackmd.io/_uploads/S1J4q5W-gl.png)
![image](https://hackmd.io/_uploads/SkD7XHoWeg.png)
![image](https://hackmd.io/_uploads/rJ5IXHsWeg.png)

---

Selanjutnya add public traffic
![image](https://hackmd.io/_uploads/rJUwNBibgg.png)
![image](https://hackmd.io/_uploads/S1h8HSi-xl.png)
![image](https://hackmd.io/_uploads/HyGiHBoZlg.png)

---

Cluster

![image](https://hackmd.io/_uploads/rJk6SSjWgx.png)
![image](https://hackmd.io/_uploads/ByHxIBj-gx.png)

![image](https://hackmd.io/_uploads/r16_LSjWeg.png)

![image](https://hackmd.io/_uploads/HkMqISjble.png)



![image](https://hackmd.io/_uploads/B1A9V5Vbge.png)

![image](https://hackmd.io/_uploads/SJEfSq4Zxl.png)

Install the desired ISO to be availabled in the cloud environment

![image](https://hackmd.io/_uploads/SyltPcEZel.png)



## Buat Instance

![image](https://hackmd.io/_uploads/HyFTYPjbxg.png)

![image](https://hackmd.io/_uploads/rJRb5wsZlg.png)

![image](https://hackmd.io/_uploads/BJjX5DsWll.png)

### Add Network untuk Instance

![image](https://hackmd.io/_uploads/BkWQOUo-le.png)

![image](https://hackmd.io/_uploads/H1SP8sNWlx.png)

![image](https://hackmd.io/_uploads/Hybwcvibel.png)

![image](https://hackmd.io/_uploads/SJf_5DiWgl.png)

Instance berhasil dibuat

![image](https://hackmd.io/_uploads/SyDi9wj-ge.png)



## Managing Cloud Resource Through CLI
Ada opsi lain untuk memanage resource cloudstack seperti menggunakan Cloudmonkey.

### Installing Cloudmonkey
```
snap install cloudmonkey
cloudmonkey --version
```
```
cloudmonkey set url http://192.168.1.50:8080/client/api
cloudmonkey set apikey zYisITlcuVSz6W7trDM3qBwigI5JqfUoj5Yu27FJCswHqHn6slZLk1jd6RzWLiuLT-2tpR-U3Oijg4JWyk3MtQ

cloudmonkey set secretkey GkacHmbpIZvyyYRIIwHNWR-rMaj7UwHcy--aQRaIG5FUZmav56nWYIlf6B_wYbR7HG7Y9j9CQkzogx5D2se0AA
cloudmonkey sync

```