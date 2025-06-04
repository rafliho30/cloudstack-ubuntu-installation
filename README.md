# Apache CloudStack: Private Cloud Installation

![Logo-Teknik-Elektro-Mini-1-1024x461-1](https://hackmd.io/_uploads/ByoVk42-ee.png)

**Members**:
1. Rafli Adithia - 2206026523
1. Darren Adam Dewantoro - 2206816600
1. Darmawan Hanif - 2206829175
1. Kevin Raihan - 2206059704

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

### Specifications

- CPU : Intel Core i5 gen 8
- RAM : 24 GB
- Storage : 250GB
- Network : Ethernet 100GB/s
- Operating System : Ubuntu Server 24.04

## Pre-installation
### Setting up SSH
First we install net-tools and ssh to be able to config our server remotely

```bash
sudo apt update
sudo apt install net-tools openntpd openssh-server sudo vim tar -y
sed -i '/#PermitRootLogin prohibit-password/a PermitRootLogin yes' /etc/ssh/sshd_config
sudo vim /etc/ssh/sshd_config
```
Find the line 'PermitRootLogin' make sure it set to 'yes'. Make sure that its uncommented

![image](https://hackmd.io/_uploads/r1njZ9-Wge.png)
```bash
sudo systemctl enable ssh
sudo systemctl start ssh
sudo ufw allow ssh
```


Next we check our ip address using **ifconfig**
```bash
ifconfig
```
We get the IP address 192.168.xx.xx and the netmask of our network /xx
To connect to our server we can use the following command on our device : 
```bash
ssh kelompok18@[IP_ADDRESS]
```

### Installing hardware resource monitoring tools
```bash
sudo apt install htop lynx duf -y
sudo apt install bridge-utils
sudo apt-get install intel-microcode -y
```

### Configure network
```bash
cd /etc/netplan
sudo nano ./0*.yaml
```
The opened file should look similarly like this:
![image](https://hackmd.io/_uploads/B1V3gqZbxx.png)

You can configure the IP address to static or dhcp4 for the cloudbr0. But be wary when you change the connected network it won't work properly anymore.

### Configure LVM
```bash
#not required unless logical volume is used
lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
resize2fs /dev/ubuntu-vg/ubuntu-lv
```

## Cloudstack Installation

```bash
sudo -i
mkdir -p /etc/apt/keyrings 
wget -O- http://packages.shapeblue.com/release.asc | gpg --dearmor | sudo tee /etc/apt/keyrings/cloudstack.gpg > /dev/null
echo deb [signed-by=/etc/apt/keyrings/cloudstack.gpg] http://packages.shapeblue.com/cloudstack/upstream/debian/4.18 / > /etc/apt/sources.list.d/cloudstack.list
```

![image](https://hackmd.io/_uploads/H1jpfcWbeg.png)
```bash
apt-get update -y
apt-get install cloudstack-management mysql-server
```

```
nano /etc/mysql/mysql.conf.d/mysqld.cnf
```
![image](https://hackmd.io/_uploads/rkcmXcZZxl.png)

```bash
cloudstack-setup-databases cloud:cloud@localhost --deploy-as=root:kelompok18 -i localhost
```
Configure Primary & Secondary Storage
```bash
apt-get install nfs-kernel-server quota
echo "/export  *(rw,async,no_root_squash,no_subtree_check)" > /etc/exports
mkdir -p /export/primary /export/secondary
exportfs -a
```
Configure NFS Server
```bash
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
```bash
apt-get install qemu-kvm cloudstack-agent -y
```

```bash
sed -i -e 's/\#vnc_listen.*$/vnc_listen = "0.0.0.0"/g' /etc/libvirt/qemu.conf

# On Ubuntu 22.04, add LIBVIRTD_ARGS="--listen" to /etc/default/libvirtd instead.

sed -i.bak 's/^\(LIBVIRTD_ARGS=\).*/\1"--listen"/' /etc/default/libvirtd
```

### Restart Libvirtd
```bash
systemctl mask libvirtd.socket libvirtd-ro.socket libvirtd-admin.socket libvirtd-tls.socket libvirtd-tcp.socket
systemctl restart libvirtd
```
### Configuration to Support Docker and Other Services
```bash
echo "net.bridge.bridge-nf-call-arptables = 0" >> /etc/sysctl.conf
echo "net.bridge.bridge-nf-call-iptables = 0" >> /etc/sysctl.conf
sysctl -p
```

### Generate Unique Host ID
```bash
apt-get install uuid -y
UUID=$(uuid)
echo host_uuid = \"$UUID\" >> /etc/libvirt/libvirtd.conf
systemctl restart libvirtd
```

### Configure Iptables Firewall and Make it persistent
```bash
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
```bash
ln -s /etc/apparmor.d/usr.sbin.libvirtd /etc/apparmor.d/disable/
ln -s /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper /etc/apparmor.d/disable/
apparmor_parser -R /etc/apparmor.d/usr.sbin.libvirtd
apparmor_parser -R /etc/apparmor.d/usr.lib.libvirt.virt-aa-helper
```

### Launch management Server
```bash
cloudstack-setup-management
systemctl status cloudstack-management
tail -f /var/log/cloudstack/management/management-server.log #if you want to troubleshoot 
#wait until all services (components) running successfully
```

### Open Web Browser and Change Password
```bash
http://<YOUR_IP_ADDRESS>:8080
```
![image](https://hackmd.io/_uploads/BkHCFqbZgx.png)

Login dengan default username
```bash
username : admin
password : password
```
Kita akan mengubah password default menjadi : 
```
username : admin
password : kelompok18
```
![image](https://hackmd.io/_uploads/S1J4q5W-gl.png)
### Add a Zone
Next, we will add a new Zone within the CloudStack environment. The configurations to be added are:
```
Name : ZONE-1
IPv4 DNS1 : 8.8.8.8
IPv4 DNS2 : 1.1.1.1
Internal DNS1 : 192.168.1.50
```
- IPv4 DNS indicates the IP addresses of public DNS servers that will be used within the zone for external domain name resolution.
- Internal DNS is the IP address of the internal DNS server used by instances within the zone for internal domain name resolution.
#### Whats a zone?
A zone is the largest organizational unit that is a collection of one or more “Pods” (each containing a host and a primary storage server) and a secondary storage server shared by all pods within the zone. Zones provide physical isolation and redundancy
![image](https://hackmd.io/_uploads/SkD7XHoWeg.png)
We will also configure:

- Hypervisor: The virtualization software used to run VMs (KVM).
- Guest CIDR: The IP address block (CIDR) for the guest (VM) network within this zone (10.1.1.0/24).

![image](https://hackmd.io/_uploads/rJ5IXHsWeg.png)

---

### Add public traffic
Here we define the range of public IP addresses that virtual machines (VMs) in your cloud will use to access the internet. The details for each IP range : 
- Gateway: The IP address of the gateway for this public IP range (192.168.1.1).
- Netmask: The subnet mask associated with this IP range (255.255.255.0).
- VLAN/VNI: (Optional) The VLAN or VNI ID if your network uses VLANs or overlay networks.
- Start IP & End IP: The beginning and end IP addresses of the public IP range to be allocated (192.168.1.241 to 192.168.1.250).
![image](https://hackmd.io/_uploads/rJUwNBibgg.png)
### Add a Pod
We add the first pod to the zone and configure a range of reserved IP addresses for CloudStack's internal management traffic. These IPs are crucial for the management plane to communicate with hosts and other components within the pod. The following details are provided: 
- Reserved system gateway: The gateway IP for the reserved system IPs (192.168.1.50).
- Reserved system netmask: The subnet mask for the reserved system IPs (255.255.255.0).
- Start reserved system IP & End reserved system IP: The starting and ending IP addresses of the range reserved for CloudStack's internal management (192.168.1.235 to 192.168.1.240).
#### Whats a Pod?
A logical grouping of hosts and primary storage servers within a Zone. Each pod contains a collection of compute and storage resources.
![image](https://hackmd.io/_uploads/S1h8HSi-xl.png)
### Select Guest Traffic
In this phase, we specify a range of VLAN IDs or VXLAN network identifiers (VNIs) dedicated to carrying communication between end-user virtual machines (VMs). This provides network isolation for individual guest networks. The following detail is provided:
- VLAN/VNI range: A numerical range defining the available VLAN IDs or VNIs for guest networks (1000 to 3000).
![image](https://hackmd.io/_uploads/HyGiHBoZlg.png)

---

### Add Cluster
We add the first cluster to the pod. A cluster serves as a logical grouping for hosts, which will be added later.
![image](https://hackmd.io/_uploads/rJk6SSjWgx.png)
### Add a Host to the Cluster
In this phase, we add the first host (physical computer) to the cluster. To enable the host to function in CloudStack, we provide its network credentials so the CloudStack management server can connect to it. The following details are provided:
- Host name: The DNS name or IP address of the host (192.168.1.50). This is how CloudStack will identify and connect to the host.
- Username: The username for accessing the host (root). This account needs appropriate permissions for CloudStack to manage the host.
- Authentication Method: The method used to authenticate with the host, either Password or System SSH Key.
- Password: The password for the specified username on the host.
#### Whats a Host?
A physical server (computer) where guest virtual machines (VMs) will run. It must have hypervisor software installed and be connected to the CloudStack management server.
![image](https://hackmd.io/_uploads/ByHxIBj-gx.png)
### Designate Primary Storage to Cluster
In this phase, we add the first primary storage server to the cluster. Primary storage is essential as it contains the disk volumes for all virtual machines (VMs) running on hosts within this cluster. We configure its details as follows:
- Name: A logical name for the primary storage (PRIMARY).
- Scope: Defines the scope of the primary storage, typically Zone or Cluster.
- Protocol: The storage protocol used to access the primary storage (nfs). CloudStack supports various standards-compliant protocols.
- Server: The IP address or hostname of the primary storage server ( 192.168.1.50).
- Path: The specific path on the storage server where the disk volumes will reside (/export/primary).
- Provider: The storage provider used (DefaultPrimary).
#### Why do we need Primary Storage
 Provides disk volumes for running VMs. This is where the actual virtual hard disks of your VMs are stored and accessed during operation. It is typically high-performance storage.
![image](https://hackmd.io/_uploads/r16_LSjWeg.png)
### Designate Secondary Storage
Secondary storage is crucial for storing VM templates, ISO images, and VM disk volume snapshots. This server must be accessible to all hosts within the zone
![image](https://hackmd.io/_uploads/HkMqISjble.png)
Finally after configurating our Zone we launch it to our server
![image](https://hackmd.io/_uploads/B1A9V5Vbge.png)
![image](https://hackmd.io/_uploads/SJEfSq4Zxl.png)

---
### Install the desired ISO to be available in the cloud environment
What we Installed : 
- VMware Tools: A suite of utilities that enhances the performance of the virtual machine's guest operating system and improves virtual machine management.
- XS Tools: Similar to VMware Tools, but specifically for XenServer (now Citrix Hypervisor), providing enhanced performance and management capabilities for VMs.
- Ubuntu 22.04 LTS: A specific version of the Ubuntu Linux operating system, with LTS standing for Long Term Support, indicating extended maintenance and support.
![image](https://hackmd.io/_uploads/SyltPcEZel.png)
___
### Add a new instances
We must first select the specific infrastructure components within the CloudStack environment where the new virtual machine (instance) will be deployed. This involves choosing the desired logical and physical location such as Zone, Pod, Cluster, and Host.
![image](https://hackmd.io/_uploads/HyFTYPjbxg.png)
We select the ISO for ubuntu server jammy 22.04 
![image](https://hackmd.io/_uploads/rJRb5wsZlg.png)
We define the computational resources and storage capacity for the new virtual machine.
![image](https://hackmd.io/_uploads/BJjX5DsWll.png)

### Add Network untuk Instance
We configure the network details for the new virtual machine, ensuring it can communicate within the cloud and potentially with external networks. Here we create a local network within the cloud with the isolated network option named (NETWORK).

Isolated network means a network that is logically isolated from other networks. Guests (VMs) in the Isolated Network can communicate with each other using private IP addresses. Each isolated network has its own virutal router which is automatically provided by Cloudstack. This virtual router will manage guest traffic for VMs (guests) when accessing outside networks.

First determine the name and type of network offering, namely “Offering for Isolated Networks with Source Nat service enabled”, zone selected domain and others.

![image](https://hackmd.io/_uploads/BkWQOUo-le.png)

then add the gateway and netmask. **The inputted gateway and netmask will determine the private IP address neighborhood available within the Isolated Network**. Add the DNS as well.
![image](https://hackmd.io/_uploads/H1SP8sNWlx.png)

In the isolated network configuration for this guest, it also needs to be configured so that it can ping out. Egress rules can be configured as needed, for example here the destination is the default route.
![image](https://hackmd.io/_uploads/HJ3akBdfex.png)

On the ip connected to the virtual router (isolated network created earlier), an option is also added to the firewall where traffic from any origin is not blocked by the firewall.

![image](https://hackmd.io/_uploads/H1rMZN_Glx.png)

Then, click OK.

Next, in the add new instance option, we can select the NETWORK that was created earlier.
![image](https://hackmd.io/_uploads/Hybwcvibel.png)

![image](https://hackmd.io/_uploads/ry017ruGee.png)

![image](https://hackmd.io/_uploads/SJf_5DiWgl.png)

When the options are all in accordance with those in the picture, click Launch Instance.

Instance successfully made

![image](https://hackmd.io/_uploads/SyDi9wj-ge.png)

These instances can be accessed via the View Console button or web browser if connected to the same local network or from outside if configured on your local router network.

![image](https://hackmd.io/_uploads/B1BVmr_Mee.png)

![image](https://hackmd.io/_uploads/ryu3NdjWee.png)

Then continue the installation until you get to the main terminal of the ubuntu CLI (or other distros).

![image](https://hackmd.io/_uploads/BysV3oaMeg.png)

![image](https://hackmd.io/_uploads/BkQQu3pzlx.png)

Next you can do port forwarding on port 22 for the private port and any public port (ex: 10010) to the public ip address that is the NAT Source from the Network tab -> Public IP addresses -> IP that is the NAT source

![image](https://hackmd.io/_uploads/By1b8o6Mlg.png)

Add the previously created VM as a port forwarding target
![image](https://hackmd.io/_uploads/r1k4Ljaflg.png)

![image](https://hackmd.io/_uploads/ryTDUjpfle.png)

![image](https://hackmd.io/_uploads/Syb9nipGxx.png)

After port forwarding is done we can access the vm that is created in isolated network using ssh

```
ssh -p 10010 kelompok18vm@192.168.1.243
```
![image](https://hackmd.io/_uploads/r1b232TGgg.png)

---
### Managing Cloud Resource Through CLI
There is another way to manage cloud resource using Cloudmonkey

#### Installing Cloudmonkey
```bash
snap install cloudmonkey
cloudmonkey --version
```

```bash
cloudmonkey set url http://192.168.1.50:8080/client/api
cloudmonkey set apikey zYisITlcuVSz6W7trDM3qBwigI5JqfUoj5Yu27FJCswHqHn6slZLk1jd6RzWLiuLT-2tpR-U3Oijg4JWyk3MtQ

cloudmonkey set secretkey GkacHmbpIZvyyYRIIwHNWR-rMaj7UwHcy--aQRaIG5FUZmav56nWYIlf6B_wYbR7HG7Y9j9CQkzogx5D2se0AA
cloudmonkey sync

```

## References

- Apache Cloudstack 4.18 Documentation: https://docs.cloudstack.apache.org/en/4.18.0.0/index.html
- https://github.com/AhmadRifqi86/cloudstack-install-and-configure/tree/main

### YouTube Link :
**https://youtu.be/Xi4Z2mnE2wY**