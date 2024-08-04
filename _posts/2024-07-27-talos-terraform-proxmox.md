---
layout: post
title: "Deploying a Talos Linux K8s cluster on Proxmox using Terraform"
tags: talos kubernets proxmox terraform
---

I have slowly building up my Terraform-on-proxmox chops, and my next challenge was to build a kubernetes cluster via Terraform. I had done something similar at one of my previous jobs, where I was building code to bootstrap RKE2 k8s nodes into Nutanix and VxRail (vSphere) virtual environments.  

I had done [previous work]({% post_url 2024-07-25-vyos-terraform-proxmox-notes %}) on deploying a vyos router on terraform, and like before, I was going to have to incrementally improve my existing terraform vm-producing code to support features that my kubernetes nodes need. My terraform repo: <https://github.com/kodama-labs/tf-proxmox>

### Updating Terraform Module

The main thing that I needed to add in was a way to inject a cloud-init user-data script into the vm provisioning process. Before this, I was just using the Proxmox built-in method of specifying a desired username, password, and ssh key(s) for provisioning. Proxmox behind the scenes would then create a cloud-init drive with a generated user-data file for your VM to consume. 

If you want to do more than just those three things though, you will need to define your own custom user-data file. However, there was some bad news and good news. The bad news is that proxmox does not yet support supplying a custom user-data file via their API, ONLY via cli... the good news is that the `bpg` terraform provider that I was using leveraged SSH to overcome this limitation, and using their VM provider I can specify my own user-data file.

This was great! There were only a few minor things I had to do:

- Enable "snippets" on the storage device that I wanted to put my user-data scripts
- Upload a user-data script to my storage

For the second thing, I could again leverage a feature of the `bpg` terraform provider to do this:

```
resource "proxmox_virtual_environment_file" "user-data" {
  content_type = "snippets"
  datastore_id = "local"
  node_name = "my-pve-node"
  source_raw  {
    data = file("local/path/my_user_data.yaml")
    file_name = "my_user_data.yaml"
  }
}
```

There was another little problem I had to address, though, and that was that for the `bpg` VM terraform provider code, the `user_account` and  `user_data_file_id` fields were mutually exclusive, and if you had both defined, then the `user_account` field would take preference. I had some default values in my `user_account` block that would always get defined even if we didn't want it, so I just wrapped my `user_account` block in a dynamic block to conditionally prevent it from showing up.

```
dynamic user_account {
      # will create this block once if user_data_file_id is null
      for_each = var.user_data_file_id != null ? [] : [""]
      content {
        keys = var.pubkey != null? [var.pubkey] : null
        password = random_password.password_resource.result
        username = var.username
      }
    }
```

Besides that, the only other minor thing I had to do was to add a variable for the default ip gateway of my VM's NICs. With that, I had everything I needed to deploy my kubernetes cluster, at least from a terraform perspective.

[`main.tf` before module updates](https://github.com/kodama-labs/tf-proxmox/blob/a7c76fb7a99397cce91b13892a60dfbc4831f2ca/vm/main.tf)

[`main.tf` after module updates](https://github.com/kodama-labs/tf-proxmox/blob/b6436734063cac9e427aae5fddb53b4f62c56bf5/vm/main.tf)


## Configuring Talos Linux

Like I had mentioned before, I had some experience bootstrap-deploying RKE2 clusters. I also had some prior experience in my homelab manually deploying a k3s cluster. Why switch vendors? Well, I saw a lot of other people (i.e. reddit comments) saying good things about Talos, how it was super easy to upgrade/maintain and had a lot of security by design of it being an immutable distribution. It's been on my periphery for a while now, so I figured I would give it a shot.

The fact that I chose Talos was the reason why I had to update my Terraform code to support custom cloud-init configuration. If it was any other distro, I would have likely opted to use Ansible for the actual post-provisioning config steps, but Talos does not support ssh (a conscious security decision by the company who makes the distro).

Instead of ssh or some other mechanism, the only way to connect to a talos node for management purposes is HTTP/HTTPS over port 50000. For doing this and other things, there is a command-line utility called `talosctl`.

### Getting the talos linux images

The first thing I needed to do was get disk images for talos. The website has specific instructions for proxmox, but since I am using terraform with cloud-init, I needed to follow the instructions for nocloud. This was important because the proxmox instructions use a `metal` image, which I believe does not have cloud-init on it. Instead I downloaded the specific `nocloud` images.

The talos `nocloud` images are packaged as `*.raw.xz`. The `bpg` proxmox terraform provider has options for specifying disk compression, but `xz` is not one of those options. Also, proxmox itself requires that the file extension be `.img` or `.iso`. 

So to fix both these issues I did something along the lines of:

```
mv talos-nocloud.raw.xz talos-nocloud.img.xz
unxz talos-nocloud.img.xz
# should result in talos-nocloud.img being in your directory
```

### Generating talos cluster config

The other thing I needed before setting up the cluster was a set of config files to distribute to my future control plane and worker nodes. This can done using `talosctl`. After installing `talosctl` on my machine, I ran the following:

```
TALOS_CONFIG_DIR=/path/to/my/desired/config/dir
CONTROL_PLANE_IP=10.10.X.X
talosctl gen config test-cluster https://$CONTROL_PLANE_IP:6443 --output_dir $TALOS_CONFIG_DIR
```

This command created a `controlplane.yaml`, `worker.yaml`, and `talosconfig` file in a directory of my choosing. The `controlplane.yaml` is used as the user-data file for the control plane nodes, and same same for the `worker.yaml` file.

With all that set up, I could upload the yaml files as snippets to proxmox and the talos image as well, and I was good to go!

### Troubleshooting issues

... or so I thought. The very first time I deployed a node, I was using the `metal` images by mistake. This meant that none of my proxmox cloud-init config, which included networking settings and user-data, got applied to the machine. By using a console session I was at least able to manually configure ip and gateway settings, and I was able to verify connectivity via ping and `netcat` (checking port 50000). I could have gone further and ran `talosctl apply-config` commands, but I wanted to do that with my cloud-init user-data file instead.

So I found and downloaded a `nocloud` image and reran my terraform scripts. This time, my network configuration made it to the VM, but I was getting different errors pop up on the VM serial console:

```
(paraphrasing)
Failed to load config via platform nocloud
...
unknown keys found
config line XYZ invalid
```

Ping still worked, but `netcat` indicated that port 50000 was closed or otherwise unavailable.

This confounded me for a while because I could find no evidence of this issue online. Eventually I realized that I had just downloaded an old talos image! My talos image version was 1.6.x, and my `talosctl` version was 1.7.x. After switching my image to a 1.7.x variant of `nocloud`, this error was resolved.

### Network Connectivity Issues

Finally I was able to run some basic commands like `talosctl dashboard` and `talosctl health` to see info about my cluster without resorting to my serial VM connection. The next step in the `nocloud` configuration is to run

```
export TALOSCONFIG="$TALOS_CONFIG_DIR/talosconfig"
talosctl config endpoint $CONTROL_PLANE_IP
talosctl config node $CONTROL_PLANE_IP
```

and then run `talosctl bootstrap`. Unfortunately here is where it was getting stuck again, saying it was waiting for etcd to start and never getting past that. Luckily though, now I could run `talosctl dmesg | less` to dig through logs, and searching for `etcd` related events led me to a series of errors relating to the `etcd` container image having issues downloading.

This led me to find some issues specific to my particular home networking setup that was not allowing https traffic out to the internet for these nodes (accidentally allowed udp/443 instead of tcp/443 on one of my firewall rules). After I fixed that and waited a bit, my single node cluster finally came up and I was finally able to run the long-awaited commands without errors:

```
talosctl kubeconfig
kubectl get nodes
```

## Scaling up

So i have a single node kubernetes cluster. What's so good about that? It's not that great, but since I provisioned it in terraform it's straightforward to increase my counts.

My original terraform code looked like this

```
module "test-talos" {
    source = "git::https://github.com/kodama-labs/tf-proxmox.git//vm?ref=talos"
    has_qemu_agent = false
    node_name = "my-pve-node"
    datastore_id = "local-lvm"
    cpu_cores = 2
    memory_gb = 2
    disk_size = 10
    tags = ["terraform", "talos", "test"]
    on = true
    hostname = "talos-test"
    networking = [
      { ipv4="10.10.X.X", ipv4_gateway="10.10.X.1"}
    ]
    template_file_id = proxmox_virtual_environment_file.talos_nocloud.id
    user_data_file_id = proxmox_virtual_environment_file.talos_config.id
}
```

Note that I am using a git source for my module. the `//vm` specifies that I want the module in the `vm` subdirectory of my git repo, and the `?ref=talos` means I want the branch that matches the reference `talos` (in this case that is just my branch name). This is all documented on the terraform website here: <https://developer.hashicorp.com/terraform/language/modules/sources>. I was using the `talos` branch because I was updating my terraform module code at the same time; the `talos` branch has since then been merged and deleted.

To scale, I can add a count field and some local variables:

```
locals {
  talos_ips = ["10.10.X.5/24", "10.10.X.6/24", "10.10.X.7/24"]
}
module "test-talos" {
    count = length(local.talos_ips)
    source = "git::https://github.com/kodama-labs/tf-proxmox.git//vm?ref=talos"
    has_qemu_agent = false
    node_name = "my-pve-node"
    datastore_id = "local-lvm"
    cpu_cores = 2
    memory_gb = 2
    disk_size = 10
    tags = ["terraform", "talos", "test"]
    on = true
    hostname = "talos-${count.index}"
    networking = [
      { ipv4=local.talos_ips[count.index], ipv4_gateway="10.10.X.1"}
    ]
    template_file_id = proxmox_virtual_environment_file.talos_nocloud.id
    user_data_file_id = proxmox_virtual_environment_file.talos_config.id
}
```

Running this would get me three nodes, and if I wanted more I could add more ips to my `local.talos_ips` variable. And I could do a similar thing to create several worker nodes as well,just need to create another iterated module that points to the `worker.yml` instead for user-data. Note that if I expected to need to change the order of my ips or insert into the middle of my `talos_ips` array, it might be better to leverage `for_each` for iteration instead of `count`.

## Next Steps

Now what? Well, this is all in my "dev" proxmox environment (an old dell laptop), and I want to get some final things working before I can confidently push a cluster out to my "prod" environment (two old thinkpads and a NAS-sized desktop, all running proxmox):

1. configure MetalLB on the cluster and have it do iBGP peering to a vyos router, which is in turn doing eBGP peering to my top-of-home pfSense router
2. That vyos router is currently manually configured, need to write ansible playbook to script that out 
2. I need a persistent storage mechanism for my k8s cluster. I want to try using proxmox CSI plugin with ceph, which would hopefully allow my k8s workloads to make persistent volume claims against a proxmox-level ceph pool.