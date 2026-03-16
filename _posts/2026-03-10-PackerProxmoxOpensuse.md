---
title:  "Proxmox and Packer - automate OpenSuse installation using ISO"
layout: post
---

During a meeting with a friend, we had a talk about snapshots (which went around VMware, ZFS, BTRFS, storage arrays snapshots…). At the end, OpenSuse came up, that by default uses BTRFS as filesystem when is installed. This means we can natively use snapshots, and of course this can be beneficial for any machine (VM or physical), from fast rollback until backup. In this first post I to digged a little bit in the installation of OpenSuse to check the BTRFS subvolumes creation for each important mountpoint and later investigated how the installation can be automated. For the next post, I will investigate how snapshots can be used for backups.
<!--more-->

In my homelab I mostly use Ubuntu cloud images and cloud-init to apply to all VMs the same standard that I defined. My first step was manually install OpenSuse and later figuring out how to automate this. This path took me to Packer, which perhaps is more harder way for this goal. As equivalent to cloud-init, previous versions of OpenSuse use YaST for automation of the OS setup, but Leap (current OpenSuse version) uses Agama. Agama is the new Suse installer, and if you want to know more about it check this: [agama](https://agama-project.github.io/about)

Packer is widely used for example to create VM golden images, which is an image with an operating system and a defined baseline in terms of software, packages and security. This image is then used mostly for immutable VMs, where instead of updating the VMs, the VMs are entirely replaced by new ones with the updated software and other baselines that were defined. 

For be able to start a VM installation with Packer, we must first define which builder we need to use, which in my case is the “Proxmox ISO”. With this builder I can start the installation of a VM using an ISO, in Proxmox (for Proxmox, there's also a second builder that is used to clone a VM template). Then, using for example the "Ansible" provisioner, I can customize this installed OS. In my case, I will simply install OpenSuse and define a user for some day-2 operations. Packer allows inject keystrokes commands when the machine starts, simulating a human typing in the keyboard that navigates in the installation process. In my case, I will define the keystrokes that will point the boot to load a JSON file which will do the OS unattended installation (*sequence in boot_command*).


```hcl
packer {
  required_plugins {
  name = {
      version = "~> 1"
      source  = "github.com/hashicorp/proxmox"
    }
    ansible = {
      version = "~> 1"
      source = "github.com/hashicorp/ansible"
    }
  }
}


source "proxmox-iso" "suse-agama" {
    boot_command= [
    "<down><wait>", 
    "e",
    "<down><down><down><down><end>",
    "<spacebar>inst.auto=http://{{ .HTTPIP }}:{{ .HTTPPort }}/autoinst.json",
#    " inst.cmdline=\"agama.profile=http://{{ .HTTPIP }}:{{ .HTTPPort }}/autoinst.json\"",
    "<spacebar>agama.debug=1",
    "<f10>"
    ]
    boot_key_interval = "50ms"
  boot_wait    = "5s"
  disks {
    disk_size         = "30G"
    storage_pool      = "nvme"
    type              = "scsi"
    discard            = "true"
    
  }
  http_directory           = "config"
  http_interface            = "Wi-Fi" # this will force the http server run with ip address of the Wi-fi, since packer binds to the first interface (in my case is a physical interface which has no IP, resulting in suse not able to get the autoinst.json file)
  http_port_min = 8111
  http_port_max = 8111
  insecure_skip_tls_verify = true
  boot_iso {
    iso_file                 = "local:iso/Leap-16.0-offline-installer-x86_64-Build171.1.install.iso" 
    iso_checksum       = "sha256:d780a228058fb4688a31b5bde22d7b97c86e04a2ced33f3d7c8e88a710d96d24"
  }
  network_adapters {
    bridge = "vmbr0"
    model  = "virtio"
  }
  scsi_controller = "virtio-scsi-single"
  machine = "q35"
  qemu_agent = "true"
  node                 = "proxmox_node_name"
  password             = "proxmox_password"
  proxmox_url          = "https://PROXMOX_IP_OR_FQDN/api2/json"
  template_description = "OpenSuse Leap, generated on ${timestamp()}"
  template_name        = "testsrv-tmp"
  username             = "root@pam"
  ssh_username         = "ansible_automation"
  ssh_timeout          = "20m"
  ssh_private_key_file = "ansible_automation_private_key"
  memory = 2048
  cpu_type = "host"
}

build {
  sources = ["source.proxmox-iso.suse-agama"]
}

```
When Packer starts, for the Proxmox builder, if we define the http_directory property, a HTTP server is instantiated and can be used to serve the files for the unattended installation. Since the details about this HTTP server are unknown, Packer allows the use of the variables :{{ .HTTPIP }}:{{ .HTTPPort }} to point the boot to the correct HTTP server that has the JSON file that will start the unattended installation.
The autoinst.json file is below, where some of the fields are blank due to security reasons. The scripts section is used to craete the 2-day operations user and add the allowed commands in its sudoers file. The storage section was kept as default, since it separates the important mountpoints by individual BTRFS subvolumes: 

```hcl
{
    "product": {
        "id": "openSUSE_Leap"
    },
    "root": {
        "password": "root_hashed_512_password", 
        "sshPublicKey": "ssh_key"
    },
    "user": {
        "userName": "username",
        "fullName": "fullname",
        "password": "password",
        "hashedPassword": false
    },
    "hostname": {
        "static":"testsrv"
    },
    "localization": {
        "language": "",
        "keyboard": ""
    },
    "software": {
        "patterns": [
            "base",
            "enhanced_base",
            "sw_management"
        ],
        "packages": [
            "vim",
            "htop",
            "curl",
            "qemu-guest-agent",
            "git",
            "tree",
            "traceroute",
            "net-tools",
            "alloy",
            "podman",
            "tar"
        ]
    },
    "network": {
        "connections": [
            {
                "id": "enp6s18",
                "interface": "enp6s18",
                "method4": "manual",
                "addresses": [
                    "192.168.0.100/24"
                ],
                "gateway4": "192.168.0.1",
                "nameservers": [
                    "192.168.0.1"
                ],
                "autoconnect": true
            }
        ]
    },
    "scripts": {
        "post": [
        {
            "name": "create_automation_user",
            "url":"http://192.168.0.64:8111/create_user.sh"
        },
        {
            "name": "create_sudoers_file",
            "url":"http://192.168.0.64:8111/ansible_automation.sh",
            "chroot": true
        }
        ]
    }
}
```

You may ask why I used a script file to add the sudoers file, and not the [Files](https://documentation.suse.com/sles/16.0/html/SLES-x86-64-agama-automated-installation/index.html#files-configuration-agama-installation-profile) module of Agama. The reason is because the scripts run after the Files, and adding the file to sudoers.d directory, results in error since the user is not yet created.

Files used for creating the user and sudoers file:
**create_user.sh**
```bash
#!/bin/bash

USER=username
KEY="ansible_automation"

useradd -m -s /bin/bash "$USER"

install -d -m 700 -o "$USER" -g "$USER" /home/$USER/.ssh
echo "$KEY" > /home/$USER/.ssh/authorized_keys

chown "$USER:$USER" /home/$USER/.ssh/authorized_keys
chmod 600 /home/$USER/.ssh/authorized_keys
```

**ansible_automation.sh**
```bash
#!/bin/bash

touch /etc/sudoers.d/ansible_automation
echo "Cmnd_Alias APT_CMDS = /usr/bin/zypper refresh,/usr/bin/zypper update" >> /etc/sudoers.d/ansible_automation
echo "Cmnd_Alias ALLOY_CMDS = /usr/bin/tee /etc/alloy/config.alloy,/usr/bin/curl" >> /etc/sudoers.d/ansible_automation
echo "Cmnd_Alias MONITORING_REPO_CLONE = /usr/bin/git" >> /etc/sudoers.d/ansible_automation
echo "ansible_automation ALL=(root) NOPASSWD: APT_CMDS, ALLOY_CMDS, MONITORING_REPO_CLONE" >> /etc/sudoers.d/ansible_automation

```

Important links:
- [Hashed password](https://documentation.suse.com/sles/16.0/html/SLES-x86-64-agama-automated-installation/index.html#root-authentication-agama-installation-profile)
- [Script section](https://documentation.suse.com/sles/16.0/html/SLES-x86-64-agama-automated-installation/index.html#scripts-configuration-agama-installation-profile)