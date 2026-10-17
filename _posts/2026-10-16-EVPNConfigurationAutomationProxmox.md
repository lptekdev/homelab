---
title:  "Automate EVPN infrastructure in Proxmox SDN"
layout: post
---

In one of my previous post, I have an example of how can I have a multitenant setup using Proxmox SDN. I decided to write this post to show how this EVPN setup can be automated, focusing only in just one tenant, by using Terraform and Ansible to automate the setup of VyOS router and the Proxmox EVPN controller and Zone. The code is located [here](https://github.com/lptekdev/Automate-EVPN-set-in-Proxmox-and-with-VyOS), if you wish to jump over the post. Before running is good to read the documentation and adjust the mentioned variables.

<!--more-->

As you may know, Proxmox offers a SDN solution based in VXLAN and EVPN as control plane, but alike NSX with its edge routers for north-south traffic, to have the similar setup, an external router can be used for the same purpose. As the external router I will use VyOS, but to automate the entire process and allow this router to be customized at the startup with the cloud-init, is required to build the image and include the cloud-init package (I also included the qemu-guest-agent). The generic architecture is below:


![EVPN setup](../assets/EVPN_automation.png)
As you see in the previous image, there's dedicated VTEP subnet and IPs on it configured in Proxmox hosts and VyOS router (in my case is VLAN 50).

To make this setup simple, the VyOS router will have 3 network interfaces:
-	Management (for Ansible access)
-	VTEP for the overlay
-	Home LAN (this interface will be configured as DHCP to receive an IP address from my home network router, and perform SNAT, to allow workloads communicate to internet)


The first part of the process is to create an ISO file with the VyOS and cloud-init package, because with cloud-init I can setup a basic configuration on the VyOS router that allows me run the Ansible playbooks later ([VyOS cloud-init](https://docs.vyos.io/en/1.5/automation/cloud-init.html#vyos-config-commands-key)). After checking the following docs: [build-iso](https://docs.vyos.io/en/rolling/contributing/build-vyos.html#build-iso) and [build-vyos](https://docs.vyos.io/en/rolling/contributing/build-vyos.html#id5) , I decided to use a docker host where I can download the base rolling image of VyOS and then build it to include the required packages. In the docker container I run the command:

```
./build-vyos-image --architecture amd64 --build-by "some_name" generic --custom-package "qemu-guest-agent cloud-init"
```

This builds an ISO file that I will use to install VyOS in a VM in Proxmox to then create a template from it. After install VyOS using this custom ISO, and without any network adapter in the hardware settings, I issued the command: ```cloud-init clean --logs``` to clear the cloud-init status and to allows run it again on the startup when the template is cloned.

The second part is using Terraform to create the VyOS router based on the previous template. The Terraform will also generate the 3 MAC addresses required for the 3 NICs on the VyOS, because those are also used to set the interfaces in the VyOS operating system, since when VyOS boots might change the order of the NICs and as also its associated VLAN, causing the IP addresses be configured in the wrong interfaces. It will also run a cloud-init prepared *vyos_config_commands* that will setup an IP address for the management interface. At the end, creates the EVPN controller and the SDN Zone on Proxmox.

Below the *vyos_config_commands* injected by cloud-init during the boot of VyOS, to configure the management IP interface and enable the SSH service:
```
    data = <<-EOF
    #cloud-config
    vyos_config_commands:
    - set interfaces ethernet eth0 hw-id '${macaddress.eth0.address}'
    - set interfaces ethernet eth1 hw-id '${macaddress.eth1.address}'
    - set interfaces ethernet eth2 hw-id '${macaddress.eth2.address}'
    - set system host-name '${var.router_hostname}'
    - set interfaces ethernet eth0 address '${var.management_ip_address}'
    - set protocols static route 0.0.0.0/0 next-hop '${var.management_gw}'
    - set service ssh
    - set service ssh disable-host-validation
    EOF
```

The last part is run the ansible playbooks to finish the configuration of the VyOS router. These playbooks isolate the management interface in its own VRF and configure the remaining settings in the VyOS to allow it to participate in the EVPN network among the Proxmox hosts to then route the traffic to outside of the SDN. 

Run the commands in the following sequence:
```
1. ansible-playbook -i inventory.ini -e ansible_user=vyos -e ansible_password=PASSWORD router-tenantA-add-management-vrf.yaml
2. ansible-playbook -i inventory.ini -e ansible_user=vyos -e ansible_password= PASSWORD router-tenantA-add-commands.yaml
``` 

As already shared, you can find all code [here](https://github.com/lptekdev/Automate-EVPN-set-in-Proxmox-and-with-VyOS), and the *ansible_username* and *ansible_password`* are the VyOS router credentials that you define when you install the VyOS template. By default the username is *vyos*, and in the installation is prompted for the password.


