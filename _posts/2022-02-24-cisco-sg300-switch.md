---
layout: post
title: "Configuring Cisco SG-300 Switch"
date: 2022-02-24
---


I bought a Cisco SG-300 off Newegg so that I could get more familiarity with Cisco switches. In hindsight, I wish I picked a switch that was more geared towards network professionals, because some of the commands are different and the feature set is not as big. However I still had a great learning experience using the switch.

## Initial configuration process

1. First thing I did was save username and password. it is in my password manager.

2. run this to see all the commands that make up your config:

```
show run 
```

(or from config:)

```
do show run
```

3. When I set it up for the first time, I had something in there that looked like:

```
username chkodama password encrypted <STUFF> privilege 15
```

I didn't want privilege 15 for login, so I created a root login with 15 and recreated chkodama with privilege 7.

4. Set enable password: this lets me do elevated stuff when logged in as chkodama

```
enable password level 15 <PASSWORD>
```

5. initial setup of ssh (password based):

```
line ssh
password <PASSWORD>
exit
```

6. So far my config looks like:

```
hostname kodama-switch                                
line ssh
password <HASH> encrypted
exit
enable password level 15 encrypted <HASH>
username root password encrypted <HASH> privilege 15
username chkodama password encrypted <HASH> privilege 7
```

7. copying config: 

```
copy running-config startup-config 
```

8. So i have some basic config stuff now but I want to be able to ssh into my switch, and there's a bunch of stuff I don't understand. using https://networklessons.com/cisco/ccna-200-301/configure-ssh-cisco-ios

```
hostname switch
ip domain name kodama.local
crypto key generate rsa
```

9. ran the command:

```
show ip interface
```

and got the hint that the ip address of the switch was likely 192.168.1.254. However when I tried to ssh into it, I got an error about incompatible ssh key ciphers. Tried updating firmware to fix this.


```
# sees version of stuff
do show version 
```

9. (b) later I changed the ip address by assigning it to a vlan with:

```
conf t
int vlan 29
ip address 192.168.42.104 /24 192.168.42.1
```

10. however, version looked fine! looks like i'm just stuck using bad ciphers. This is possible via:

```
ssh -oKexAlgorithms=+diffie-hellman-group1-sha1 chkodama@192.168.1.254
```

11. copied to startup config again.

12. disabled all ports:

```
int range ge1-20
shut
exit
```

13. enabled some ports and configured them as access:

```
int ge1
no shut
switchport mode access
exit
```

14. copied run to start again

```
copy run start
```

15. importing an ssh public key

```
conf t
crypto key pubkey-chain ssh
user-key chkodama rsa
key-string
<paste in just the stuff after ssh-rsa and before user@host>
```

16. connecting to it remotely with pubkey

```
ssh -o KexAlgorithms=+diffie-hellman-group1-sha1 -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedKeyTypes=+ssh-rsa -i id_rsa chkodama@192.168.42.104 -p6922
```
  
  
# Serial connection to switch

I had some issues getting the right cable for the job. After some trial and error, I was able to use a serial cable daisy-chained to a serial-to-usb adapter and that worked for me. 

First, installing minicom 
```
apt-get install minicom
pacman -S minicom # arch
```

Then I ran:
```
dmesg | grep tty
[    0.225265] printk: console [tty0] enabled
[    0.841190] 0000:00:16.3: ttyS4 at I/O 0x1800 (irq = 17, base_baud = 115200) is a 16550A
[  513.835830] usb 3-1: FTDI USB Serial Device converter now attached to ttyUSB0
```

This shows I need to use ttyUSB0

Then I ran:

```
sudo minicom -s
```

and followed the directions in: https://www.ismoothblog.com/2019/07/access-cisco-switch-serial-console-linux.html

Basically: 

1. Set the serial path to /dev/ttyUSB0
2. set the rate to 115200 8N1
3. Set hardware flow control to no
4. saved setup as cisco

Then I ran:

```
sudo minicom cisco
```