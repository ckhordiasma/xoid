---
layout: post
title: "Setting up Ceph on Proxmox"
tags: proxmox ceph kubernetes
---

This all started because I wanted a way to be able to create persistent volume claims on locally hosted kubernetes cluster. I originally had longhorn installed on my cluster for this, but I wanted to try something else. I looked into a project that implemented a proxmox CSI for kubernetes, but it seemed to have some limitations on migration and seemed intimidating to install. Ceph on the other hand, had a nice blue button on the proxmox UI that I could click to try, and it seemed easy enough to at least dip my toes in the water. I didn't have anything important running on my cluster at the moment (which is why I was able to just delete longhorn without issue), so it was a good time to experiment.

I have two proxmox clusters at home:

### "dev" cluster 

- a single dell latitude laptop - 16GB RAM - ~800GB local storage + 500GB SSD

### "prod" cluster 

- Lenovo W530 laptop - 24GB RAM - ~500GB local storage + 1TB SSD 
- Lenovo W510 laptop - 24GB RAM - ~500GB local storage + 500GB SSD
- custom desktop in a 8-bay NAS form factor - 64GB RAM - ~800GB local storage + several ZFS pools

All networking is currently just gigabit, which is way smaller than the 10Gbps that the ceph docs recommend, but our cluster is small so far so maybe it will be ok.

## Networking

My proxmox nodes are connected to a managed switch via trunked ports with a native vlan. I am also using the proxmox SDN feature to keep vlan tags consistend across my nodes. I use the native vlan for proxmox management and all other vlans off the trunk (maybe not a good idea to use the native vlan for management? a TODO for later)

During the initial configuration prompt for Ceph, it asks what network to put ceph on, and the only option it gave me was my native vlan. I wanted to keep this ceph stuff a little more separate from my other stuff in case things go sideways, so I created a new linux VLAN (vmbr0.XX) for ceph on each node. I did all this stuff on my dev cluster before trying on the prod one.

## Troubleshooting 

When doing it on my prod cluster, after creating these new vlans and entering them into the ceph install wizard, I got an error saying that the network interface was not found. This was... because I forgot to apply the network config on proxmox when I added the new linux VLAN. This ended up leaving the ceph installation in a weird half-configured, broken state, and I had to do some googling to fix it. What worked for me was some combination of:

- running `pveceph purge`
- running `systemctl restart ceph.target`
- DELETING `/etc/pve/ceph.conf` 

The first and last command were risky moves that I was willing to do because my ceph cluster had nothing on it.

## Object Storage Daemons (OSDs)

The OSDs are the workers that perform the actual ceph storage operations. The proxmox docs said to have one OSD per disk as a best practice. The ceph docs also said not to have OSDs on your operating system (in my case, my proxmox debian OS) disks. With that in mind, I installed one OSD for the extra disks on my Thinkpads (installed where the CD Drive bay used to be). 

## Ceph Pools

A pool is kinda like a zpool, it's a logical storage pool that is redundant and can span multiple devices. I made one called "cesspool" with all the default storage options. 

## First impressions

Just to test it out, I migrated my idle kubernetes clusters to the ceph pool, and then tried migrating my nodes around between proxmox nodes. Migrations were nice and fast even on my gigabit connection because only the VM state was being migrated. 

## More troubleshooting

I cancelled a migration from local-lvm to my ceph pool, and that caused an error with ceph that I had to google to resolve. The error was a generic

```
rbd error: rbd: listing images failed: (2) No such file or directory (500)
```

I was able to get a much better message by running `rbd ls -l cesspool` on any of my proxmox nodes, that told me which device had an issue. Then I ran `rbd rm disk-name-here -p cesspool` to delete that disk (which I did not need).

## TODO

I still have a lot of things left to do with this ceph cluster. Here are some thoughts

- Set up a third ceph node, I guess the NAS drive? But I would need to add a hard drive for it. Maybe I can use my dell laptop instead?
- set up the ceph CSI. this is going to not only require installing the CSI in my cluster, but also figuring out how my kubernetes can talk to ceph. right now ceph is isolated in its own LAN
- test out what happens if I shut down a proxmox node. Does everything stay up and running?
