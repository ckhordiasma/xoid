---
layout: post
title: "Logging on Proxmox"
---

I've recently ran into some situations where things on my proxmox server were failing, and I had no idea about it until later because I didn't check my proxmox frequently enough. To help resolve this, I set up a few forms of email logging on my proxmox server.

## Email Logging on Proxmox Events

Following this tutorial: 

https://www.youtube.com/watch?v=85ME8i4Ry6A
https://technotim.live/posts/proxmox-alerts/

```
apt update 
apt install libsasl2-modules mailutils
echo "smtp.gmail.com <FROM_EMAIL>:$APP_PASSWORD" > /etc/postfix/sasl_passwd
postmap hash:/etc/postfix/sasl_passwd 
chmod 600  /etc/postfix/sasl_passwd
```

add to bottom of /etc/postfix/main.cf
comment out the `relayhost =` line in the default config
```
relayhost = smtp.gmail.com:587
smtp_use_tls = yes
smtp_sasl_auth_enable = yes
smtp_sasl_security_options = 
smtp_sasl_password_maps = hash:/etc/postfix/sasl_passwd 
smtp_tls_CAfile = /etc/ssl/certs/Entrust_Root_Certification_Authority.pem
smtp_tls_session_cache_database = btree:/var/lib/postfix/smtp_tls_session_cache
smtp_tls_session_cache_timeout = 3600s
```

then run

```
postfix reload
```

to test: 

```
echo "test message" | mail -s "hello world" <TO_EMAIL>
```

to change username of sent emails:

```
apt install postfix-pcre
echo "/^From:.*/ REPLACE From: proxmox-alert <FROM_EMAIL>" > /etc/postfix/smtp_header_checks
postmap hash:/etc/postfix/smtp_header_checks
```
next, edit /etc/postfix/main.cf and add to the botom:

```
smtp_header_checks = pcre:/etc/postfix/smtp_header_checks
```

finally, make sure your root account in datacenter -> users -> root is your real email address that you want to recieve alerts from.

# emailing syslogs using logcheck

`logcheck` can be used to send a filtered subset of your syslogs to email. I wanted to have it check logs for only one program I was using on my proxmox host: `zfsup`.  After installing, I edited a few things

1. cron - by default the cronjob for logcheck runs 2 minutes past every hour. I changed mine to `0 2 * * 2`, which is 2AM on tuesdays. The cron file is located at `/etc/cron.d/logcheck`
2.  logcheck search for zfsup - logcheck has three sections that use regexes differently. the `/etc/logcheck/violations.d` and `/etc/logcheck/cracking.d` folders _search_ for specific regexes, with the intent of putting key words indicating bad stuff going on. I added a `zfs_uploader` file in the `violations.d` folder with the content: 

```
^.*zfsup.*$
```

which would make sure that all logs that reference zfsup would be reported on (as a "violation")

3. logcheck extra ignore - now that I had logcheck doing what I wanted. I set it up to ignore some additional logs I was getting. to do this i created a file `/etc/logcheck/ignore.d.server/local` and put in some regexes:

```
^(\w{3} [ :0-9]{11}|[0-9T:.+-]{32}) [._[:alnum:]-]+ pmxcfs[2150]: [status] notice: received log.*$
^(\w{3} [ :0-9]{11}|[0-9T:.+-]{32}) [._[:alnum:]-]+ pmxcfs[2150]: [dcdb] notice: data verification successful.*$
```