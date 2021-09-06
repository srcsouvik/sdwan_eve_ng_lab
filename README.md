# Cisco SDWAN EVE-NG Lab
In this lab, we will go through how to configure CISCO SD-WAN lab in EVE-NG. Which include the following

* Initial topology for the control plane devices: vManage, vBond, and vSmart.
* vEdge and cEdge onboarding
* Replicate ZTP onboarding process
* Extending by replicating real world scenario's with small, medium large type of site or branch offices
* TLOC extension for Medium sites
* Working with device and feature templates
* Enable control policies to achieve hub-n-spoke type topology*
* Prefer one transport over other
* Replicate migration from Non-SDWAn sites to SDWAN site/routing between SDWAN and Non-SDWAN sites.
* Enable local internet breakdown and failover to DC default route

We will be working with the below topology:
![image](https://user-images.githubusercontent.com/84218572/132181134-dfcaa57b-21ac-48dd-b837-c493e8f957ae.png)

# 1. Software/hardware used in this lab
## 1.1. Lab Software
Here are the list of softwares used in this lab:
* EVE-NG Community Edition 2.0.3-112
* vManage - 20.4.2
* vBond and vEdge - 20.4.2
* vSmart - 20.4.2
* vEdge - 20.4.2
* cEdge(CSR1000v) - 17.03.03
* Cisco IOL l2/l3 - Version 15.2d/15.5(2)T
## 1.2. EVE-NG system requirements
In this lab EVE-NG was setup in Google Cloud Platform(GCP) with 32 GB of RAM and 16 vCPU's allocated to it, for more on how to setup EVE-NNG in GCP refer the EVE-NG Cookbook documentation section 3.4 page 43. 

* vManage is 12 GB RAM, 2 vCPUs, 30 GB storage
* vSmart is 4 GB RAM, 1 vCPU, no required storage
* vBond is 2GB RAM, 1 vCPU, no required storage
* The other nodes:
* Border Router: CSR1000v - 3GB RAM, 1 vCPU
* 2 vEdges router: 2 GB RAM and 1 vCPU each

When the full lab is running the gsn3 VM CPU 28.5%; RAM 85.9%.
## 1.3. Control Plane setup
First thing in SDWAN is to bring up the control plane for this we will focus on the below section of the topology

![image](https://user-images.githubusercontent.com/84218572/132176819-1ec4b037-af62-4c05-8426-6744d17268ac.png)

Host | VPN 0 (control) | VPN 512 (mgmt)
---- | --------------- | --------------
vManage |	eth0 - 1.1.1.1/28	| eth1 - 192.168.1.1/24
vSmart | eth0 - 1.1.1.2/28	| eth1 - 192.168.1.2/24
vBond	| Ge0/0 - 1.1.1.3/28 |	eth0 - 192.168.1.3/24
Linux | eth0 - 1.1.1.4/28 |  eth1 - 192.168.1.4/24

Here the Linux host is being used as the CA server and also to gain GUI access to the vmanage for configuration purpose.

# 2. Control plane devices configuration
Initial bootstrap configuration of the controllers

## 2.1. vManage
Skinny config for vManage:

```
config
system 
host-name vManage
system-ip 100.1.1.1
site-id 1
organization-name srcsdwanlab
vbond 1.1.1.3
vpn 0
 interface eth0
  ip address 1.1.1.1/28
  no shutdown
  tunnel-interface
   allow-service all
   allow-service sshd
   allow-service netconf
 ip route 0.0.0.0/0 1.1.1.14
vpn 512 
 interface eth1
  ip address 192.168.1.1/24
  no shutdown
commit and-quit
```
## 2.2. vSmart
Skinny config for vSmart:

```
config
system
 host-name vSmart
 system-ip 100.1.1.2
 site-id 1
 organization-name srcsdwanlab
 vbond 1.1.1.3
vpn 0
 interface eth0
  ip address 1.1.1.2/28
  no shutdown
  tunnel-interface
   allow-service all
   allow-service sshd
   allow-service netconf
 ip route 0.0.0.0/0 1.1.1.14
vpn 512 
 int eth1
  ip add 192.168.1.2/24
  no shutdown
commit and-quit
```
## 2.3. vBond
Skinny config for vBond:

```
config
system
 host-name vBond
 system-ip 100.1.1.3
 site-id 1
 organization-name srcsdwanlab
 vbond 1.1.1.3 local
vpn 0
 interface ge0/0
  ip address 1.1.1.3/28
  no shutdown
  tunnel-interface
   allow-service all
   allow-service sshd
   allow-service netconf
 ip route 0.0.0.0/0 1.1.1.14
vpn 512
 interface eth0
  ip address 192.168.1.3/24
  no shutdown
commit and-quit
```
## 2.4. Linux VM setup as CA Server
Next we launch the Linux VM and using a terminal window generate the Root CA key and certificate, that will be used to sign the controller certificates.
Command to Generate the keys and CA cert:
```
openssl genrsa -out CA.key 2048
openssl req -new -x509 -days 1000 -key CA.key -out CA.crt
```
Transfer the CA cert generated above to the controllers using SCP:
```
scp CA.crt admin@192.168.1.1:
scp CA.crt admin@192.168.1.2:
scp CA.crt admin@192.168.1.3:
```
Install the root CA on the controllers:
```
request root-cert-chain install /home/admin/CA.crt 
```
# 3 Adding Controllers to the vManage
Now we connect to the vManage controller from the Linux VM from a browser on the vpn512 interface IP, https://192.168.1.1 or https://192.168.1.1:8443. Upon login first thing we should do is set the Organisation name and vBond IP by navigating to ```Administration > Settings```.
