---
layout: post
title: Configuring hibernate on linux laptop
tags: linux systemd
---

I recently got a new laptop with arch linux on it, and I was noticing that if I forgot to plug my charging cable for the night and came back a day later, I would lose a big chunk of battery charge overnight. My laptop enters the sleep state when idle, and by running `cat /sys/power/mem_sleep` I can see the sleep states that are compatible with my system... in my case, I only have an `s2idle` option and nothing else.

To fix my power consumption issue, I wanted to set my computer to hibernate instead of going to sleep when idle. The arch wiki has a good power article about properly configuring hibernate. I already had a swap partition set up, so I just needed to edit my `mkinitcpio.conf` to add `resume` as a hook, and then run `mkinitcpio -P` to reload. 

Then I ran into an issue... If I ran `systemctl hibernate`, my screen would go dark for a few seconds but then turn back on and return me to a login screen. Once in a while though (I tried hibernating several times), the hibernate would work correctly. Running `journalctl -b` helped me find some issues after I searched for "hibernate":

```
3 20:15:35 my-pc kernel: xhci_hcd 0000:c1:00.3: PM: pci_pm_freeze(): hcd_pci_suspend returns -16
3 20:15:35 my-pc kernel: xhci_hcd 0000:c1:00.3: PM: dpm_run_callback(): pci_pm_freeze returns -16
3 20:15:35 my-pc kernel: xhci_hcd 0000:c1:00.3: PM: failed to freeze async: error -16
# and then later
3 20:15:35 my-pc systemd-sleep[14620]: Failed to put system to sleep. System resumed again: Device or resource busy

```

So the pci device `0000:c1:00.3` was causing my hibernate to fail. Running lspci indicated that the culprit was `c1:00.3 USB controller: Advanced Micro Devices, Inc. [AMD] Device 15b9`. 

## Disabling power wakeup for specific PCI device

I could disable the wakeup capabilities of this pci device by running the following command:

```
echo disabled > /sys/bus/pci/devices/0000\:c1\:00.3/power/wakeup
```

This worked great! My hibernations were working consistently. The only problem was that this setting went back to `enabled` on every reboot, and so if I wanted to fix it this way I would have to add scripts of some sort to disable it on boot or something. I wanted to avoid doing that if possible, so I looked for other options.

## Configuring suspend-then-hibernate

After looking at the manpage for systemd-sleep, I noticed that there was an interesting setting called suspend-then-hibernate. This would sleep for a configurable amount of time, and then if still idle, would wake up and immediately hibernate. I liked this for a few reasons:

- if my laptop is only idle for a short amount of time, I would prefer the immediate wakeup from sleep over a hibernate
- maybe if the hibernate happens after already being asleep, I won't run into the pci error from before.

To implement this, I made a folder/file called `/etc/systemd/sleep.conf.d/hibernate.conf` (`hibernate.conf` is my arbitrary name for this config)

```
[Sleep]
AllowHibernation=yes
AllowSuspendThenHibernate=yes
HibernateMode=shutdown
HibernateDelaySec=10s # set to 10s for testing purposes
```

Then I ran `systemctl suspend-then-hibernate` ... and it worked! Hooray!

Then I started applying it to actual events like my laptop lid closing by making a file/folder called `/etc/systemd/logind.conf.d/hibernate.conf`

```
[Login]
HandleLidSwitch=suspend-then-hibernate
```

However, when I tried closing my lid, nothing happened! My computer went back to a login screen but otherwise nothing. What was going on?

## GNOME overrides

Well, it turns out that GNOME was overriding my power events; I could see a list of inhibited stuff via `systemd-inhibit`. Maybe I could find a gnome setting to do suspend-then-hibernate when idle? Well, it turns out that this is not a thing in GNOME anymore, it was for a while but then got reverted back apparently.

Instead though, I found some GNOME settings that were using sleep (the default) instead of hibernate on idle, and I changed those to hibernate. I did these with a dconf editor gui but the commands to do it would look like this:

```
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-battery-type 'hibernate'
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-battery-timeout 300

# This setting useful for testing out what happens when idle, but the default of 300s is decent I think
gsettings set org.gnome.desktop.session idle-delay 10
```

## Disable GNOME power overrides

I didn't do this, but I thought it was worth mentioning. If I was dead set on implementing suspend-then-hibernate, then I would have to do something to disable the GNOME power settings. There were some inhibitor override settings in `/etc/systemd/logind.conf` that might work, or disabling the power plugin somehow from within the gnome settings themselves.

# Conclusion

I ended up going the GNOME settings route, and have my computer hibernate after 10 minutes: 5 minutes before being considered idle, at which point the screen will shut off. 5 minutes after that, the computer will hibernate. I think this will give me some nice big power savings when i forget to charge my laptop overnight!

## Epilogue

Well, it turns out that entrusting gnome to properly hibernate was a bad assumption! gnome would sometimes silently fail to hibernate my computer, and I would see the same error messages that I was getting before in `journalctl -b`. So I changed up my settings to test alternatives:

```
# set to a short value for testing purposes
gsettings set org.gnome.desktop.session idle-delay 10

gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-battery-timeout 0
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-battery-type 'nothing'
```

My `/etc/systemd/logind.conf.d/hibernate.conf` file:

```
[Login]
HandleLidSwitch=suspend-then-hibernate
LidSwitchIgnoreInhibited=yes
# how many seconds to wait after lid close to initiate HandleLidSwitch event
#  kept it short for testing purposes
HoldoffTimeoutSec=10s

IdleAction=suspend-then-hibernate
# how many seconds to wait after idle to initiate IdleAction
#  kept short for testing purposes
IdleActionSec=5s
```

and `/etc/systemd/sleep.conf.d/hibernate.conf`:

```
[Sleep]
AllowHibernation=yes
AllowSuspendThenHibernate=yes
HibernateMode=shutdown
HibernateDelaySec=300s
```

With these settings, GNOME will go idle after ten seconds, and then after 5 seconds, systemd will initiate suspend-then-hibernate. After 5 minutes of suspend, the system will go into hibernation. I tested it a few times and it worked! Also, the settings at the top of the logind script say to put the system in suspend-then-hibernate after the lid has been closed for 10 seconds.

After doing all this testing, I set everything to more normal values:


```
gsettings set org.gnome.desktop.session idle-delay 300

gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-battery-timeout 0
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-battery-type 'nothing'
```

`/etc/systemd/logind.conf.d/hibernate.conf`:

```
[Login]
HandleLidSwitch=suspend-then-hibernate
LidSwitchIgnoreInhibited=yes
# how many seconds to wait after lid close to initiate HandleLidSwitch event
HoldoffTimeoutSec=20s

IdleAction=suspend-then-hibernate
# how many seconds to wait after idle to initiate IdleAction
IdleActionSec=30s
```

and `/etc/systemd/sleep.conf.d/hibernate.conf`:

```
[Sleep]
AllowHibernation=yes
AllowSuspendThenHibernate=yes
HibernateMode=shutdown
HibernateDelaySec=300s
```

## Conclusion 2

So now I have a new power configuration that should save me battery life! I am still not sure if this is going to work the best. For example, if I am running a long script or download, will my system go idle during it? That might cause me headaches down the road. As a workaround I could set 

```
gsettings set org.gnome.desktop.session idle-delay 999999
```

to some large value (or maybe 0 disables it?) and prevent GNOME from idling. If gnome never goes idle, then systemd should not hibernate my system. I guess I will see what happens!
