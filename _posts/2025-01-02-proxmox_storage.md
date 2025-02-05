---
title:  "Proxmox Homelab - Part 2 - Adding storage as a service"
layout: post
---

Welcome to the second part of my Proxmox Homelab. In this post, we add a new service to the multitenancy infrastructure: external storage. At the end there is the configuration required for this setup.
One of the services that is used by the Tenants is storage. Although VMs use block storage, sometimes it is required to have centralized storage that can be shared by multiple workloads. 
<!--more-->

Two alternatives come up with this: use an external storage system that Tenants don’t need to manage, or virtualize a storage appliance within the Tenant environment, having Tenants full control of it (meaning also they need to handle the management).

I have decided to follow the first approach, setting up an external storage system. The easiest way to accomplish this is by routing the storage traffic through the Tenant firewall, but this can have a big impact on the workloads.
The other way, which is not so easy, is routing the traffic through routers R1 and R2, and then to another router: R3, which acts as a bridge to the storage system. By using BGP, Tenant address space and storage subnet routes are exchanged between these routers.

On the storage side, there will be a switch and a Truenas server, both with VLANs support. The R3 will have configured a sub-interface (is called pseudo interface in VyOS) that connects to Truenas sub-interface (in the same VLAN), using a /30 subnet (creating a point-to-point connection). Each Tenant will have a dedicated /30 subnet if they need to consume this service. This subnet is in the address space: 172.16.3.0-172.31.255.255, resulting in 261,952 subnets.

Of course we have the limitation of the 4094 VLANs, but since every storage system is isolated in a switch and each Tenant has its own sub-interface in R3 (meaning that we may have another network interface with the same sub-interface VLAN ID), we can scale horizontally the switch and the Truenas system, increasing also the capacity of the service to more Tenants (within the previously mentioned address space capacity, and the routers interface throughput/processing power). Also, like in the Proxmox multitenancy setup, each Tenant will have its own VRF in R3, which will receive by BGP the route summary of the Tenant address space and will contain the sub-interface that connects to the storage system.

A BGP connection is established between R1 and R2, allowing R3 to receive the Tenant address space: 10.0.0.0/16. This route will be identified by the its RD: AS:Tenant_ZONE_ID, allowing R3 tom import it to the correct VRF routing table. The route that is only advertised from R3, to R1 and R2 is the subnet that connects to the storage system.

The Proxmox servers, despite being in the same subnet and same AS of the R3, are not configured as BGP neighbors with R3. As we saw in part 1 of this setup, every Tenant has configured in R1 and R2:
* a sub-interface that connects to the perimeter firewall to allow outside communication
* default route through this firewall, which is announced to the Proxmox servers and injected in the Tenant VRF routing table

This means that even without this BGP connection from R3 and the Proxmox servers, the workloads will be able to reach the storage through the next-hop defined by this route, which is R1 and R2 (since the default route includes the storage subnet). By the split horizon rule, the storage subnet route will never be advertised to the Proxmox servers, so we need to rely on the default route to guarantee there’s path to the storage. Also, without this BGP configuration it is not expected that R3 receives the type 5 and type 3 EVPN routes from Proxmox servers.

Now at the storage side, a point-to-point connection is made from R3 to Truenas within an isolated VLAN, which is different for each Tenant. Using the capability in Linux: Source based routing, is possible to create multiple route tables for each Tenant in Truenas system. This route tables contains always the same destination (the Tenant address space), and by being able to coexist in the Truenas system, allow Tenant traffic isolation.
The configuration for this is below (for two Tenants: A and C):
```bash
echo "3 vlan3_table" >> /etc/iproute2/rt_tables
ip rule add from 172.16.3.2 lookup vlan3_table
ip route add 10.0.0.0/16 via 172.16.3.1 dev vlan3 table vlan3_table

echo "4 vlan4_table" >> /etc/iproute2/rt_tables
ip rule add from 172.16.3.6 lookup vlan4_table
ip route add 10.0.0.0/16 via 172.16.3.5 dev vlan4 table vlan4_table
```

In "3 vlan3_table" is represented the priority and the routing table name. Is in this table that will be stored the routes. 
The next two commands allow to add the route to the Tenant address space:
1. 172.16.3.2 – is the Truenas sub-interface IP dedicated for the Tenant
2. 10.0.0.0/16 – is the Tenant address space
3. 172.16.3.1 – is the IP address of the R3 sub-interface in the same subnet of the Truenas sub-interface in 1.

As it’s possible to see we have two different subnets for two different Tenants.
Using NFS, one big issue (and very big) that was seen, was that a "showmount -e TENANT_IP_TRUENAS" command will show all the shares available in Truenas. This means all Tenants will be able to see all Tenants shares. However, by the source based routing and VRF isolation, they will only be able to access to its own shares.


**Routers Configuration**
```bash
#R3
# general settings and BGP neighbors R1 and R2
set vrf name management
set vrf name management table 100
set interfaces ethernet eth0 vrf management
set interfaces ethernet eth0 address dhcp
set service ssh vrf management
set system host-name r3
set interfaces ethernet eth1 address 172.16.0.3/23
set interfaces ethernet eth1 description 'proxmox nodes'
set interfaces ethernet eth1 mtu 1600
set protocols bgp system-as 64513
set protocols bgp address-family l2vpn-evpn
set protocols bgp address-family l2vpn-evpn advertise ipv4 unicast
set protocols bgp address-family l2vpn-evpn advertise-all-vni
set protocols bgp neighbor 172.16.0.1 address-family l2vpn-evpn
set protocols bgp neighbor 172.16.0.1 remote-as 64513
set protocols bgp neighbor 172.16.0.1 update-source eth1
set protocols bgp neighbor 172.16.0.2 address-family l2vpn-evpn
set protocols bgp neighbor 172.16.0.2 remote-as 64513
set protocols bgp neighbor 172.16.0.2 update-source eth1

#VRF TenantA
set vrf name tenantA table 5000
set vrf name tenantA vni 5000
set vrf name tenantA protocols bgp system-as 64513
set vrf name tenantA protocols bgp address-family ipv4-unicast redistribute connected
set vrf name tenantA protocols bgp address-family l2vpn-evpn advertise ipv4 unicast

# configure tenantA pseudo-interface to storage
set interfaces pseudo-ethernet peth2 source-interface eth2
set interfaces pseudo-ethernet peth2 vif 3
set interfaces pseudo-ethernet peth2 vif 3 description StorageTenantA
set interfaces pseudo-ethernet peth2 vif 3 address 172.16.3.1/30
set interfaces pseudo-ethernet peth2 vif 3 vrf tenantA

# tenantA bridge and vxlan interface
set interfaces vxlan vxlan5000 mtu 1550
set interfaces vxlan vxlan5000 parameters nolearning
set interfaces vxlan vxlan5000 port 4789
set interfaces vxlan vxlan5000 source-address 172.16.0.3
set interfaces vxlan vxlan5000 vni 5000

set interfaces bridge br5000 description tenantA
set interfaces bridge br5000 member interface vxlan5000
set interfaces bridge br5000 vrf tenantA


# R1 configuration, configure R3 as neighbor setting also the RD for tenant A
set protocols bgp neighbor 172.16.0.3 remote-as 64513
set protocols bgp neighbor 172.16.0.3 address-family l2vpn-evpn
set vrf name tenantA protocols bgp parameters router-id 172.16.0.1
set vrf name tenantA protocols bgp address-family l2vpn-evpn rd '64513:5000'
set vrf name tenantA protocols bgp address-family l2vpn-evpn route-target import '64513:5005'

# R2 configuration, configure R3 as neighbor setting also the RD
set protocols bgp neighbor 172.16.0.3 remote-as 64513
set protocols bgp neighbor 172.16.0.3 address-family l2vpn-evpn
set vrf name tenantA protocols bgp parameters router-id 172.16.0.1
set vrf name tenantA protocols bgp address-family l2vpn-evpn rd '64513:5000'

```