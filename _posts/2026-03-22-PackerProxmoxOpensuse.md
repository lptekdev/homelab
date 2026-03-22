---
title:  "Proxmox and Packer - automate OpenSuse installation using ISO"
layout: post
---

In a meeting with friend of mine, where we discussed the snapshots technology in VMware, ZFS, BTRFS and storage arrays snapshots, the default installation of OpenSuse which uses BTRFS as filesystem and the relevant mounpoints separated per different BTRFS subvolumes was also a topic that came up and got my interest. Since I mostly use Ubuntu and cloud-init, I decided then to create this post where I explore the installation of OpenSuse using its ISO and Packer. About BTRFS snapshots, you can find here an excelent article: [BTRFS Snapshots](https://fedoramagazine.org/working-with-btrfs-snapshots/).
<!--more-->

As previously mentioned, in my homelab I mostly use Ubuntu cloud images and cloud-init to apply to all VMs the same standard that I defined when I need to create new VMs (the cloud images contains already the operating system installed). Following the same mindset, the automated installation for OpenSuse lead me to Packer. Packer is widely used for example to create VM golden images, which is an image with an installed operating system and in most of the cases with a compliant/aproved set of packages and security settings. This image is then used mostly for immutable VMs, where instead of updating the VMs, the VMs are entirely replaced by new ones with the updated software and packages.

To start this process we must first define which builder we need to use. The **builder** defines the platform where the VMs or images will be created. In this case I will use [Proxmox ISO](https://developer.hashicorp.com/packer/integrations/hashicorp/proxmox/latest/components/builder/iso), since I use Proxmox platform, allowing me to use an ISO to automate the installation of a VM. Then, using for example the [Ansible provisioner](https://developer.hashicorp.com/packer/integrations/hashicorp/ansible/latest/components/provisioner/ansible), I can customize this installed OS, like deploying some application or also hardening the system. In my case, I will simply install OpenSuse and define a user for day-2 operations. However, just for curiosity, I configured the [shell provisioner](https://developer.hashicorp.com/packer/docs/provisioners/shell) to just output the hostname of the deployed machine.
Since packer allows inject keystrokes commands when the machine starts, simulating a human typing in the keyboard that navigates in the installation process, with some help of AI I defined this keystrokes that will point the boot to load a JSON file hosted in a HTTP server which will do the OS unattended installation (*sequence in boot_command*).


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
  http_interface            = "Wi-Fi" # this will force the http server run with ip address of the Wi-fi, since the HTTP server that packer starts binds to the first interface (in my case was a physical interface which has no IP, resulting in the installer not able to get the autoinst.json file)
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
  provisioner "shell" {
    inline = ["hostname"]
  }
}

```

When Packer starts, for the Proxmox builder, if we define the *http_directory* property, a [HTTP server](https://developer.hashicorp.com/packer/integrations/hashicorp/proxmox/latest/components/builder/iso#http-directory-configuration) is instantiated and can be used to serve the files for the unattended installation. Since the deep details about this HTTP server are unknown, Packer allows the use of the variables *:{{ .HTTPIP }}:{{ .HTTPPort }}* to help pointing the boot to the correct HTTP server that hosts the JSON file that will start the unattended installation.
The contents of *autoinst.json* file are below, where some of the fields are blank due to security reasons. The scripts section is used to create the 2-day operations user and add the allowed commands in its sudoers file. The storage section was kept as default, since it separates the important mountpoints by individual BTRFS subvolumes:

```json
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
            "curl",
            "qemu-guest-agent",
            "git",
            "tree",
            "traceroute",
            "net-tools",
            "tar"
        ]
    },
    "network": {
        "connections": [
            {
                "id": "enp6s18",
                "interface": "enp6s18",
                "method4": "auto",
                "autoconnect": true
            }
        ]
    },
    "scripts": {
        "post": [
        {
            "name": "create_automation_user",
            "url":"http://PACKER_HTTP_SERVER_MACHINE_IP:8111/create_user.sh"
        },
        {
            "name": "create_sudoers_file",
            "url":"http://PACKER_HTTP_SERVER_MACHINE_IP:8111/ansible_automation.sh",
            "chroot": true
        }
        ]
    }
}
```

You may ask why I used a script file to add the sudoers file, instead the [Files](https://documentation.suse.com/sles/16.0/html/SLES-x86-64-agama-automated-installation/index.html#files-configuration-agama-installation-profile) module of Agama. The reason is because the Scripts run after the Files, and adding the file to sudoers.d directory using the native File module, results in error due the user is not yet created (since via Agama only one additional user can be created).

Script files used for creating the user and sudoers file:

**create_user.sh**
```bash
#!/bin/bash

USER=username
KEY="ansible_automation_public_key"

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


Another important links:
- [Hashed password](https://documentation.suse.com/sles/16.0/html/SLES-x86-64-agama-automated-installation/index.html#root-authentication-agama-installation-profile)
- [Script section](https://documentation.suse.com/sles/16.0/html/SLES-x86-64-agama-automated-installation/index.html#scripts-configuration-agama-installation-profile)
- [Agama RAW](https://raw.githubusercontent.com/agama-project/agama/refs/heads/SLE-16/rust/agama-lib/share/profile.schema.json)