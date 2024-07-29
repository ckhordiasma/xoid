---
layout: post
title: "Installing vyos as a Proxmox VM using Terraform"
tags: proxmox terraform vyos ansible
---

In order to get more practice with advanced networking topics, I wanted to have ways for me to quickly spin up router VMs that I can test with. After doing some googling I decided on vyos as my router of choice for now. For the "quick" part I planned on using Terraform with Proxmox (the hypervisor I use at home). 

## Terraform on Proxmox

I had some prior work with creating basic VMs and LXCs with Proxmox, but it was kind of in a disorganized state. I cleaned it up and make a public git repo with some terraform modules. This is located here: <https://github.com/kodama-labs/tf-proxmox>. I needed to make some modifications to the VM module I made to make it work for routers in general. For example, my VM module assumes the VM only needs one NIC. [here] is a snapshot of my repo from when I started this project.

Overall though, it was a good starting point.

## Cloud-init and vyos

I did not worry about the multiple NIC thing for now though. First I just wanted to see if I could get it to work by pointing my existing terraform vm module to a vyos image.

I used a nightly vyos image (nightlies are the only ones available for free) and my terraform code looked like this:

```
resource "proxmox_virtual_environment_download_file" "vyos_vm" {
  content_type = "iso"
  datastore_id = "local"
  node_name    = "my-pve-node"
  url          = "https://github.com/vyos/vyos-rolling-nightly-builds/releases/download/1.5-rolling-202407250020/vyos-1.5-rolling-202407250020-amd64.iso"
}

module "test-vyos" {
    source = "git::https://github.com/kodama-labs/tf-proxmox.git//vm"
    node_name = "my-pve-node"
    username = "vyos"
    datastore_id = "local-lvm"
    disk_size = 5
    hostname = "test-vyos"
    tags = ["terraform", "routing", "test"]
    on = true
    bridge = "LAN"
    pubkey = "my-pubkey-here"
    template_file_id = proxmox_virtual_environment_download_file.vyos_vm.id
}
```

This succesfully created and started a vyos VM! But there were issues. The main issue was that my username, hostname, and public key settings did not get transferred to the VM. I did some digging around, using searches like "proxmox terraform vyos" and "proxmox vyos cloudinit" and came across this blog post that seemed to be the right direction: <https://codingpackets.com/blog/vyos-qemu-image-build/>

The blog had instructions for downloading a vyos repo for building qcow images that was run using ansible. I set up ansible in a venv, cloned the git repo, and did the prep step of commenting out the `download-iso` step from `qemu.yml`. After that I could use this command to run the playbook:

```
sudo ansible-playbook qemu.yml \
   -e disk_size=10 \
   -e iso_local=./vyos-1.5-rolling-202407250020-amd64.iso \
   -e grub_console=serial \
   -e cloud_init=true \
   -e cloud_init_ds=NoCloud \
   -e guest_agent=qemu \
   -e enable_ssh=true
```
However, things immediately started going wrong. The playbook wants to install dependencies and mount stuff, so trying to run it directly on my computer wasn't going to work probably. Also, since it mounts an iso, using a docker container might get complicated/inconvenient, so instead I decided to do the same thing as the blog author and run the build in a vm. 

Like I mentioned earlier, i had a quick and easy way to spin up simple VMs with terraform, so I did just that. I spun up a vm, installed python, installed ansible in a venv, and did almost the same as above, but with a few changes

- did not need to sudo. ran `ansible-playbook --become` instead. 
- `qemu.yml` has a variable at the beginning for where the output qemu will be stored. I changed it to `../` instead of `/tmp`.
- after it failed the first time because of permissions issues, subsequent attempts failed until i deleted a /tmp/something.iso that got created as a mid-build step. 

Finally I had a custom shiny qcow and just had to throw it in proxmox now.

## Using a qcow in terraform 

The first thing I did was just do a manual upload of the qcow into proxmox, and tested if a vm could pick that up. I discovered during the manual upload... that proxmox wouldn't let me upload images ending in `.qcow2`. So i changed it to `.img`, uploaded, and prayed for the best.

On the terraform side, I updated my module code to point directly to this image (instead of pointing to a `download_file` resource like before)

```
    template_file_id = "local:iso/vyos-1.5-rolling-202407250020-cloud-init-10G-qemu.img"
```

Note the naming convention here: `<storage>:<folder>/<filename>`

I made that change, applied my terraform... and it worked! I was able to login using the randomly generated password from terraform, and also could use my ssh key to login. Now I could roll back and attempt to upload my disk file programmatically as well.

I deleted the vm and the image from proxmox and made a data folder in my terraform repo to hold the image. Then I added the following terraform code:

```
resource "proxmox_virtual_environment_file" "vyos_cloudinit" {
  content_type = "iso"
  datastore_id = "local"
  node_name    = "my-pve-node"

  source_file {
    path = "./data/vyos-1.5-rolling-202407250020-cloud-init-10G-qemu.img"
  }
}
```

Additionally, I updated my vyos terraform config to point to this image:

```
    template_file_id = proxmox_virtual_environment_file.vyos_cloudinit.id
```

## updating terraform vm module

Now that I know I can sucessfully provision a vm (including capability for further config with later steps, like with ansible), I needed to update my terraform vm module with a few things:

- configurable CPU
- configurable RAM
- configurable NICs
- configurable ip addresses

There are other nice-to-haves but these are the important ones for the vyos router to function as an actual router. This took a couple of hours but was overall straightforward. For the NIC and IP address configuration I had to use dynamic blocks to be able to define multiple. For example, the ip configuration:

```
 dynamic "ip_config" {
      for_each = var.networking
      content {
          ipv4 {
            address = ip_config.value["ipv4"]
          }
      }
    }
```

and the nic configuration:

```
dynamic "network_device" {
    for_each = var.networking
    content {
        bridge = network_device.value["bridge"]
        vlan_id = network_device.value["vlan"]
    }
  }
```

To make defining network config less cumbersome, I made a `networking` variable with sane defaults:

```
variable "networking" {
  type = list(object({
    bridge = optional(string, "vmbr0")
    vlan = optional(number, null)
    ipv4 = optional(string, "dhcp")
  }))
  default = [{}]
  description = "a list of all your networking devices. defaults to a single nic attached to vmbr0, no vlan, and with dhcp."
}
```

with this, if you just want a VM on vmbr0 with dhcp, you can specify `[{}]` for the networking variable and get a valid output.

I tested this all with my vyos image from before, and it seems to work!! 

## What next?

I think I am at a good stopping point here in terms of terraform development. I wanted to bootstrap vyos just enough to be able to use something else like ansible to configure the rest. This overall effort was great because it will not only help me move fast as I mess around with vyos, but I was also able to improve my existing terraform vm module as well. A lot of those changes can also carry over into my lxc module, and that is something I will be doing once I need to spin up more advanced LXCs.

For VyOS, my next steps are to 

- learn how to use vyos for basic routing and firewalling, and also more advanced bgp use cases
- build an ansible role for config management of vyos routers
- deploy a router to serve as the router above a kubernetes cluster in its own subnet

For terraform:

- build terraform module to create kubernetes nodes (i think it's about time I learned talos)
- update my lxc module to match the features of my new vm module



