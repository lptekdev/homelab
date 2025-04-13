---
title:  "Proxmox Homelab - Part 4 - Expanding Ceph capacity and improve its redundancy"
layout: post
---

Welcome to the fourth post about my Proxmox homelab: Expanding Ceph capacity and increasing the redundancy of the storage service. If you didn’t read the 3 previous posts, below is a short summary about them for awareness: 

1. [Part1](/homelab/proxmox) - Enabling multitenancy – setup an environment that can host multiple tenants completely isolated and using duplicated address space.
2. [Part2](/homelab/proxmox-part2) - Adding a NFS storage solution with Truenas – tenants might need to consume storage, and the idea was to provide it by NFS. However, I faced a high security concern caused by the "nature of NFS".
3. [Part3](/homelab/proxmox-part3) - Solving the NFS problem using Ceph – implement a multitenant storage service solution using one Ceph node.  With the help of **Linux Namespaces** and customizing the ceph public network I managed to have one ceph node that provides CephFS, while I can isolate every workload uniquely.

<!--more-->

Now that I was able to setup one Ceph host to provide CephFS to the tenant workloads, is time to expand the capacity and increase redundancy of the Storage service. For this setup I used 3 machines as also Linux namespaces and bridges. The architecture of this setup is below:

![proxmox_storage_Ceph_multinode](../assets/storageAAService-multinodeCeph.png)

The architecture does not show many differences in the configuration for the Ceph nodes when compared with the architecture in post number 3, but those are important for the overall of the setup. After understanding a bit how Ceph can work in a multitenant environment (which does not come by design of Ceph), and after also understand that SNAT is the way to identify uniquely each workload, I also understood that MON and OSD daemons need to be hosted in a shared subnet (defined in public network) that needs to be accessed by all the tenants. However, the same does not applies to the MDS daemon. This means we can set a different subnet (defining its own public network), that can be different for every tenant (remember that MDS daemon is a process that allows run metadata commands on the CephFS like: cd, ls…). As all the other daemons, the MDS will bind to the first interface that is available on its defined public network.


The idea of having a MDS subnet for each tenant is to create a network isolation among the tenants. By being different for each tenant, is not required to set IPtables rules because the tenants will not be able to reach the other tenant MDS, since the /32 IPs advertised by BGP only include the tenant /29 subnet IPs. With multiple IPs on the bridge of the Host Linux network namespace, is possible to route between the different networks of MON, OSD and the dedicated tenant MDS. Another /32 IPs that need also to be advertised by BGP are the IPs initially defined on the public network of MON and OSD daemons. These IPs need individually to be advertised because these processes (MON and OSD) bind also to an IP on the defined public network, and clients must reach directly them to authenticate and perform IO respectively.


Going back to the MDS daemon, this process can be isolated from every tenant and using a /29 subnet we have 6 IPs available to handle MDS active and passive daemons, and to be set on the bridges to perform routing (in the list of commands for the Ceph installation there's one to create an additional MDS passive daemon). Also, another requirement, is the MDS active and passive daemons needs to communicate between each other and also between the MON, meaning different IPs needs to be configured on each Ceph host. The configuration to advertise the /32 IPs instead of the /29 network, is due the fact that Ceph has its own failover mechanism and the Ceph client will forward the requests to the correct MDS daemon in case of failure. This means that advertising the /29 subnet will create issues because with BGP is not possible to understand what MDS is active and what's its associated IP address. The same applies to the IPs on the MON and OSD public subnet.


The architecture doesn’t show the Ceph cluster network, but this is a different subnet with a dedicated physical interface. Also, is not shown another physical interface on the Host Linux namespace that belongs to each Ceph host bridge, but from the issued commands you can identify it easily.

Below are the commands that I issued on the 3 Ceph hosts, and finally the commands to simply install Ceph and configure a CephFS to one tenant. Please be aware that for Ceph installation and configuration, several steps are missing (like creating the client and setting the permissions, creating the cluster, create the CephFS share).

