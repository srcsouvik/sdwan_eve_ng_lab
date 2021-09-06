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
![image](https://user-images.githubusercontent.com/84218572/132160524-0e5f918c-6af4-447c-a8f1-e1347cf0d38d.png)

1. Software/hardware used in this lab
1.1. Lab Software
* EVE-NG Community Edition 2.0.3-112
* VMware® Workstation 15 Pro
* vManage - 20.4.2
* vBond and vEdge - 20.4.2
* vSmart - 20.4.2
* vEdge - 20.4.2
* CSR1000v - 17.03.03
* Cisco IOL l2/l3 - Version 15.2d/15.5(2)T
