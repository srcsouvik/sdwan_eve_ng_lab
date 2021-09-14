# Cisco SDWAN EVE-NG Lab
In this lab, we will go through how to configure CISCO SD-WAN lab in EVE-NG. Which includes the following

* Control plane setup vManage, vBond, and vSmart.
* vEdge and cEdge onboarding
* Working with device and feature templates
* Replicate ZTP onboarding process
* Replicate real world scenario's with small, medium large type of site or branch offices
* TLOC extension for Medium sites
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
In this lab EVE-NG was setup in Google Cloud Platform(GCP) with 32 GB of RAM and 16 vCPU's allocated to it, for more on how to setup EVE-NNG in GCP refer the EVE-NG Community Cookbook documentation section 3.4 page 43 onwards(https://www.eve-ng.net/index.php/documentation/community-cookbook/).

* vManage is 12 GB RAM, 2 vCPUs, 30 GB storage
* vSmart is 4 GB RAM, 1 vCPU, no required storage
* vBond is 2GB RAM, 1 vCPU, no required storage
* The other nodes:
* Border Router: CSR1000v - 3GB RAM, 1 vCPU
* 2 vEdges router: 2 GB RAM and 1 vCPU each


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
# 3. Adding Controllers to the vManage
Now we connect to the vManage controller from the Linux VM from a browser on the vpn512 interface IP, https://192.168.1.1 or https://192.168.1.1:8443.

![image](https://user-images.githubusercontent.com/84218572/132218754-12003534-e39b-478c-be70-c9b9c34d0379.png)

Upon login first thing we should do is set the Organisation name and vBond IP by navigating to ```Administration > Settings```.

![image](https://user-images.githubusercontent.com/84218572/132219053-ef89ec56-50e6-4784-b1dc-2694276b0b1c.png)
## 3.1. Installing Root CA on vManage
Next we set the controller Certificate Authorization to Enterprise Root Certificate and upload the CA.crt file
![image](https://user-images.githubusercontent.com/84218572/132219413-d70a7e13-024c-4193-a857-a486fb78b505.png)
## 3.2. Add vSmart and Vbond to vManage
Now navigating to ```Configuration > Devices > Controllers > Add Controller ``` from drop down select vSmart and provide the IP, username, password uncheck Generate CSR and click add.
![image](https://user-images.githubusercontent.com/84218572/132220125-f2cbf845-1923-4c6e-9498-d60e4c137af4.png)
Repeat the above step for adding the vBond.
## 3.3. Generate and download CSR's for vManage, VSmart and Vbond 
Next we need to generate the controller CSR's by navigating to ```Configuration > Certificates > Controllers ``` select vManage click on the three dots and Generate CSR

![image](https://user-images.githubusercontent.com/84218572/132221327-3c97b50f-9ea0-46ad-9d8a-a4e9da08c886.png)

Download and save the file as vManage.csr in the directory where we have the Root CA.

![image](https://user-images.githubusercontent.com/84218572/132221918-ac559d6b-67ea-498d-9ab3-baa995ce246b.png)

Repeat the above steps for vBond.csr and vSmart.csr.

## 3.4. Sign the controller CSR's using CA cert
Sign the certs of the Controllers:
 ```
openssl x509 -req -in vManage.csr -CA CA.crt -CAkey CA.key -CAcreateserial -out vManage.crt -days 2000 -sha256
openssl x509 -req -in vBond.csr -CA CA.crt -CAkey CA.key -CAcreateserial -out vBond.crt -days 2000 -sha256
openssl x509 -req -in vSmart.csr -CA CA.crt -CAkey CA.key -CAcreateserial -out vSmart.crt -days 2000 -sha256
 ```
 ![image](https://user-images.githubusercontent.com/84218572/132224318-e2e42041-1747-42b8-83cd-0585b1398c82.png)

## 3.5. Install the certs 
Now we need to install the certificate by navigating to ```Configuration > Certificates > Controllers ``` select vManage and then ```Install Certificate``` on top right hand corner, then selet the vManage.crt file from the directory and click install.

![image](https://user-images.githubusercontent.com/84218572/132223309-5b12fa90-d992-4270-82ea-41e81fc97c67.png)

Repeat the above steps for vSmart and vBond, once all the certs are installed the certificate status should change to installed for all the controllers.
![image](https://user-images.githubusercontent.com/84218572/132224713-df125b40-f612-4d7e-8e2d-745349db3acf.png)

## 3.6. Uploading the WAN Edge list
Befor proceeding any further we would need valid WAN Edge list to onboard any vEdge/cEdge, this can be done by going to ``` Configuration > Devices and click Upload WAN Edge List```.
![image](https://user-images.githubusercontent.com/84218572/132226516-f4728d6a-899f-45f7-9673-3ad289062ec8.png)

Once the WAN Edge is uploaded and validated it wil show up under ```Configuration > Devices```.

![image](https://user-images.githubusercontent.com/84218572/132227665-9adff67d-80f1-4712-9437-a504edcf6039.png)

# 4. Setting up the underlay transport Internet & MPLS
In this lab topology routers GW-RTR & MPLS provide the Internet and MPLS connectivity. GW-RTR also acts as the DHCP and DNS server which will later help us in orchestrating ZTP process. The configuration of these devices are straight forward to provide IP reachability over each tranport to the controllers, the configs can be found in the config folder.

# 5. vEdge Onboarding process
Skinny config for vEdge for Site10-vEdge01
```
config
system
 host-name Site10-vEdge01
 system-ip 10.100.10.10
 site-id 10
 organization-name srcsdwanlab
 vbond 1.1.1.3
!
vpn 0
 interface ge0/0
  ip address 51.51.10.10/24
  ipv6 dhcp-client
  tunnel-interface
   allow-service all
   allow-service sshd
   allow-service netconf
  no shutdown
 !
 ip route 0.0.0.0/0 51.51.51.2
 commit and-quit
```
Now we should have reachability between the controllers and the vEdge01 over internet transport, at this point we copy the CA.crt file from our jump host to the vEdge using the command 
```
scp CA.crt admin@51.51.51.51:
````
![image](https://user-images.githubusercontent.com/84218572/133274663-05b59e93-fbfa-45a6-9774-00850139d5b7.png)

To verify CA.crt was copied go into vshell of the edge device and issue ```ls -l``` command we should see the CA.crt file here.

![image](https://user-images.githubusercontent.com/84218572/133275600-73e89410-0154-4e15-98dd-7c628de69e65.png)

Next step would be to install the Root CA using the below command on the vEdge
Install the CA Cert:
```
request root-cert-chain install /home/admin/CA.crt
```
![image](https://user-images.githubusercontent.com/84218572/133275752-2202deff-b4cd-4153-9b95-168d2e0ead59.png)

The final step would be to activate the vEdge usning the command
```
request vedge-cloud activate chassis-number <uuid> token <otp>
request vedge-cloud activate chassis-number 3feafc93-2414-f6a8-e2b8-2f39156a010f token 7fb6cf6468437f37c3054ad4cf84dab5
```
Where ```uuid=3feafc93-2414-f6a8-e2b8-2f39156a010f and otp=7fb6cf6468437f37c3054ad4cf84dab5``` are obtained from vManage GUI by navigating to ```Configuration > Devices > WAN Edge list``` and selecting any free entry from the list and generating the bootstrap config and using the UUID and token value from there in the above command.

![image](https://user-images.githubusercontent.com/84218572/133277243-2063fd88-453c-46b7-bc6f-05df03fa3e77.png)

The above steps needs to be repeated for all the vedges in the topology once done successfully all should appear in the vManage dashboard and Monitor as below:

![image](https://user-images.githubusercontent.com/84218572/133279203-dddf1015-0c74-4439-938f-56a2790f3f8c.png)

![image](https://user-images.githubusercontent.com/84218572/133279429-bb91f933-b2f1-4448-a410-f6da554b5c87.png)



