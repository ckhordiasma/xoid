---
layout: post
title: "Configuring a pfSense router/firewall with Ansible"
tags: ansible pfsense networking
---

My top-of-the-house router, the one that connects to my ISP, is a pfSense router running on netgate hardware. I've slowly modified it over the years, and now it does a lot for me:

- 6 separate interfaces/subnets and corresponding firewall zones
- DNS
- DHCP for ipv4 and ipv6
- dynamic DNS for public IP
- certificate management for home servers
- avahi mDNS reflector
- tailscale subnet routing
- BGP using FRR

All of this was done using the pfSense web UI. For better overall config management, I wanted to automate as much as I could using Ansible instead. I used the `pfsensible.core` ansible package, github repo here: <https://github.com/pfsensible/core>. 

To get it working, there were a few initial steps that I had to configure manually.

1. install sudo on pfsense. There is a sudo package that you can select and install in the pfsense web UI.
2. generate (local to your machine) an ssh key pair to use with ansible+pfsense
2. make a new user in pfsense and add the ssh public key generated prior. Add it to the admins group. I named my user `ansible`.
3. Enable ssh on pfSense. System -> Advanced -> Secure Shell. Recommend setting to only allow public keys.
3. Test it out; add the ssh key to your ssh agent and try to login with `ssh ansible@192.168.x.x`
4. Create/modify your ansible inventory file to include pfsense and some defaults:

    ```
    all:
    hosts:
        pfsense:
        ansible_host: 192.168.X.X
        ansible_ssh_user: ansible
        ansible_python_interpreter: /usr/local/bin/python3.11
        ansible_pipelining: true
    ```
   
   The interpreter line is optional, it just gets rid of a warning message about auto-discovered python versions.

   The pipelining line is also optional, but I found that it significantly sped up my ansible playbook.

5. Run a test ansible script:

    ```
    ---
    - name: Test playbook
    hosts: pfsense
    gather_facts: False
    tasks:
    - name: ping
        ping:
    ```

6. Assuming you have an inventory file default defined in your `ansible.cfg`, and you have your ssh key already added to your ssh agent, this should run with `ansible-playbook test.yml`

6. Changing settings requiring sudo

    When having a command that will require sudo (which will probably be most of them), you need to add `become: yes` to your plays or specific tasks. You can specify your `ansible` user password (that you created in the first step) in your inventory file or in a host_vars file (don't put this in source control!) or specify it on the command line with the `-K` option: `ansible-playbook your-playbook.yml -K`

    ```
    ---
    - name: Configure vlan
    hosts: pfsense
    gather_facts: true
    become: yes
    tasks:
    - name: Add vlan 55
        pfsensible.core.pfsense_vlan:
        interface: mvneta1
        vlan_id: 55
        descr: for vlan 55
        state: present
    ```

## Problems

Some of the commands kind of take a while to run, but adding the `ssh_pipelining` helped speed it up pretty well. 

Another downside is that the ansible module covers all the base pfsense features pretty well, but it doesn't yet cover a lot of the third party modules like FRR and tailscale (haproxy module is there though). 

For FRR, I tried to use the existing ansible frr collection here <https://docs.ansible.com/ansible/latest/collections/frr/frr/index.html>, but I was not able to get it to work with the pfsense installation of frr... Also, apparently if I did go that route, the pfSense web UI FRR settings would go out of sync with the actual FRR config, and I didn't want that to happen.

Overerall though the module works great and I can't wait to start building out ansible playbooks to build out the majority of my pfsense config!

## Up Next

- I have one firewall zone defined using only ansible, next would be to migrate the rest of the zones over. I have some interface groups to reduce rule redundancy, but I might not need those if I structure my data right in ansible. Also need to consider, do I want a playbook that does all my zones at once? or maybe separate playbook runs for each zone? Second option might be best with how slow the module is.

- I have web certificates managed and renewed through pfSense for my various servers at home. It would be really cool if I could have ansible playbooks that requested new certificates from pfSense and then pushed those new certs to my servers!

- DHCP - I frequently check the DHCP status page to see what ip address some newly provisioned device got, being able to automate this or even do some kind of mac address lookup would be great. Being able to manage static DHCP reservations with ansible is another thing too.

