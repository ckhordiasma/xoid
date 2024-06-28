---
layout: project
title: "Home Lab"
date: 2024-05-27
---

My "home lab" started with just a raspberry pi running Pihole. Then I started working with routers/switches for at IT/networking job, and ended up buying a managed switch and pfsense router for myself to mess around with at home. My IT/networking job turned into DevOps, and I started learning things like Terraform and Ansible.. at the same time, my coworker told me about a free hypervisor called Proxmox. Things quickly progressed from there, and now my current compute/networking inventory is:

- 2 old laptops running proxmox
- 1 8-bay NAS server running proxmox
- Cisco SG-300 managed switch
- Brocade ICX 8450 managed switch
- Ruckus R750 Wireless Access Point
- Netgate 2100 running pfSense

In terms of applications, I now have:

- pihole
- several docker hosts
- a kubernetes cluster
- immich
- komga
- samba
- MacOS VM
- home assistant
- plex

And I have a huge backlog of things I want to do in the lab... for example:

- add 10G networking to NAS server
- deploy a kubernetes cluster using terraform on proxmox
- make a proper 3+ node proxmox cluster
- set up the brocade switch only using Ansible (i just bought it recently)
- set up netbox for better inventory management
- add an ELK stack for logging/metrics
- deploy something to do CI/CD, like gitlab

It has really become quite a fulfilling hobby and there's always something interesting to do.