---
layout: post
title: Running Docker in LXC
tags: docker proxmox linux
---

these are things that I had to do to get docker working the way i want on an lxc container.

I decided to go this way because I thought bind mounting a directory would have better throughput than my previous working solution
of a samba networked mount.

## container settings on proxmox host

required features:

```
features: keyctl=1,nesting=1
```

adding bind mounts:

```
mp1: /tank/share,mp=/media/share
mp2: /tank/share2,mp=/media/share2
```

large amounts of id remapping. I had to map a bunch of ids here to account for the various users and groups that were associated with the smb share.

```
lxc.idmap: u 0 100000 995
lxc.idmap: g 0 100000 995
lxc.idmap: u 995 995 1
lxc.idmap: g 995 995 1
lxc.idmap: u 996 996 1
lxc.idmap: g 996 996 1
lxc.idmap: u 997 997 1
lxc.idmap: g 997 997 1
lxc.idmap: u 998 998 1
lxc.idmap: g 998 998 1
lxc.idmap: u 999 999 1
lxc.idmap: g 999 999 1
lxc.idmap: u 1000 1000 1
lxc.idmap: g 1000 1000 1
lxc.idmap: u 1001 1001 1
lxc.idmap: g 1001 1001 1
lxc.idmap: u 1002 101002 64534
lxc.idmap: g 1002 101002 64534
```


added to /etc/subuid and /etc/subgid:

```
root:995:7
```

## settings in container

create a local user, ensure that the id matches with the mapping that was done earlier. also gave it sudo privileges

also, I had a "share2" user group in SMB, so I made sure to create another share2 user group in the container with the 
same GID, and added my local container user ("duck") to it.

installed docker according to the docker documentation, including creating the docker group and adding duck to it

modified the following line in /lib/systemd/system/docker.service
```
ExecStart=/usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock --data-root /media/cache/docker
```

to fix error mentioned in https://forum.proxmox.com/threads/docker-failed-to-register-layer-applylayer-exit-status-1-stdout-stderr-unlinkat-var-log-apt-invalid-argument.119954/, 
create a file called /etc/docker/daemon.json with the contents

```
{"storage-driver": "vfs"}
```

```
# to apply the service config file
systemctl daemon-reload 
systemctl restart docker
```
