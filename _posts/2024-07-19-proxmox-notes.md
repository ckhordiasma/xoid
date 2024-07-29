---
layout: post
title: "Proxmox Setup and Config Notes"
---

Over the past couple of years, I have been incrementally repurposing old hardware, mostly laptops, by installing proxmox on them and then running VMs, LXCs, and containers on there. I am writing this post to consolidate some notes I had about proxmox setup. 

## Disable sleep/suspend/hibernate on laptop lid close

Need to uncomment/modify the following entries in `/etc/systemd/logind.conf`

```
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
```

Run `systemctl restart systemd-logind`

## Globally disable suspend/hibernate/sleep

I was running into an issue during power outages where my laptops would not start back up, even though they have "turn on on AC power connect" on in the BIOS. To try to get this to work better, I attempted to globally disable suspend/hibernate/sleep, but it didn't fix my problem. 

Create a folder/file at `/etc/systemd/sleep.conf.d/nosuspend.conf` with contents:

```
[Sleep]
AllowSuspend=no
AllowHibernation=no
AllowSuspendThenHibernate=no
AllowHybridSleep=no
```

Run `systemctl restart systemd-logind`

## Trigger shutdown on low battery

I ended up fixing my power outage doesn't-start-back-up issue by having my laptop proxmox nodes initiate a shutdown once their battery levels are sufficiently low. 

Make a file called `/etc/udev/rules.d/99-lowbat-rules`

```
# Shut the system when battery level drops to 9% or lower, with status equal to discharging or not charging
SUBSYSTEM=="power_supply", ATTR{status}=="Not charging|Discharging", ATTR{capacity}=="[0-9]", RUN+="/usr/bin/systemctl poweroff"
```

## Renaming proxmox node that is already part of a cluster

In this example, I am renaming a node from `old-name` to `new-name`

Follow the instructions here https://pve.proxmox.com/wiki/Renaming_a_PVE_node

For the /etc/hosts file, I started by adding the `new-name` (before the `old-name`), and then later removed `old-name`.

You also have to edit /etc/pve/corosync.conf with the `new-name`, and then possibly need to `systemctl restart corosync`

I had a vm that was trapped in a "ghost" node named `old-name` after my renaming process (if I had more than one VM there, they would have all been stuck probably). 

I was able to find the vm file config using:

```
cd /etc/pve
find | grep "<VM ID>.conf"
```

this pointed me to the /etc/pve/nodes folder, where i saw a folder there called `old-name`, and my vm file config was in the qemu-server folder in `old-name`. I moved the vm conf file to the  `/etc/pve/nodes/new-name/qemu-server` folder and then it appeared in the right spot in the web console.

However, I was not able to get it to start yet because my newly renamed node was missing a storage element. The cluster keeps track of which nodes have which storages, and I had a storage that was only mapped to `old-name`. I had to go into the cluster settings and edit that storage node to remove `old-name` and add `new-name`. After I did that, I was able to get the vm to start.
