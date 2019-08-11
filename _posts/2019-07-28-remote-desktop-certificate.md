---
layout: post
title: "Remote Desktop Certificates"
date: 2019-07-28
comment_id: 8
post_id: 9
---

Last week, I made a post about my project for command-line encryption and decrpytion of Outlook emails using a smart card. A day before I worked on that project, I was actually working on another, tangentially related project relating to encryption and openssl.

I have a desktop computer that I sometimes like to remote into. I don't do anything fancy, I just use the standard Microsoft remote desktop protocol (RDP). Anyone who uses RDP for personal use will notice that when you first try to remote in to the "server" computer, you will get a warning about certificate trust. 

After seeing that warning time after time for several years, I figured it was about time I looked into how to properly certify everything correctly. If I was able to do that, then I would be able to use the warning correctly: to notify me that something insecure might be going on (because my remote computer was not able to validate the client). 

Like most projects, I first started by using a rather unknown search tool known as Google to find some information about the problem. [this](https://gist.github.com/mtigas/952344) and [that](https://www.wikihow.com/Be-Your-Own-Certificate-Authority) were some good starting points for understanding what I needed to do.

## Basic Steps

The first thing I had to do was better understand how certificates work. Once I did that, did the following things:

1. Make a root certificate authority for myself (Root CA Public Certificate)
1. Distribute root certificate to all remote clients (i.e. iphone, other laptops, etc)
1. Use Root CA to generate a intermediate certificate for my server, including settings like a revocation list location and key/extended key usage
1. Configure my server to announce that certificate if it receives any RDP requests

## Certificates

SSL/TLS certificates are a way of creating digital structures of trust, sort of like government-issued ID cards for digital objects. People "trust" real-life ID cards because they trust the government that issued those ID cards. For certificates, the underlying basis for trust is.... math, specifically the math of really large prime numbers. 

Certificates can be used to create heirarchies of trust as well. Certificates can "vouch" for other certificates, and computers (and people) trust this system of vouching because of math as well. 

Back to the problem at hand, the first thing I needed to do was to make myself a certificate authority. This is the ultimate, top-level certificate that I will use to certify everything else. It will be the God of my certificate world. The command to do this is:

```bash
# generate a RSA key named CA.key, encrypted with a DES3 algorithm-based password, key is 2048 bits long
openssl genrsa -des3 -out CA.key 2048
```

This key, CA.key, is the biggest point of failure for my system, because all subsequent encryption is based on this key. People recommend keping this private key on a USB, locked in a physical safe somewhere.

The next thing you have to do is to 1. prepare to sign the key and 2. sign the key

A sign request is a file that contains all the 'paperwork' needed before actually doing the signature operation. For my CA.key, I did the following:

```bash
openssl req -verbose -new -key CA.key -out CA.csr -sha256
```

After that, you need to sign the key using the .key file and .csr file. In my case, since I used the default .cnf configuration file that was preinstalled with openssl, I also had to make a few folders and files beforehand.

```bash
mkdir ./demoCA/newcerts
touch ./demoCA/index.txt
touch ./demoCA/index.txt.attr
echo 1000 > ./demoCA/serial

# use v3_ca settings in openssl.cnf file
# use sha256 message digestion
# certificate expire date, YYMMDDHHMMSSZ format.
openssl ca -extensions v3_ca -verbose -out CA-signed.crt -keyfile CA.key -selfsign -md sha256 -enddate 991231235959Z -infiles CA.csr
```
It is going to ask you for passwords in a few steps of the process. Don't forget them.

I now have a .crt file, which is the root CA certificate that I will use for all subsequent certificate signing. 

## Trusting certificates

On windows, this is pretty straightforward. You double click on the .crt file and install the root CA certificate through a GUI. I plopped mine in the "Trusted Root CA" folder. On an iPhone, you need to get the .crt file on your phone somehow (email it to yourself -> add it to your files -> click on it). The phone will tell you that it has downloaded this profile, and you can get to a point where it asks you to install or delete the profile. It will initially say that the profile is untrusted. After you install it, go to settings -> general -> about -> certificate trust settings -> and enable full trust for the root CA you just installed.

Now that I have done this, my desktop computer and iphone will trust anything that is signed by the root CA that I just made.

## Signed Certificate for RDP server

With my shiny new root CA, I can make a subordinate certificate (think: ID CARD) for my server. This ended up being a little tricky because of all the little nitnoid settings that I had to get correct.

```bash

# generating private key
openssl genrsa -des3 -out server.key 2048

## generating signing request for key (no config file). I ended up NOT doing this and using a config file
openssl req -verbose -new -key server.key -out server.csr -sha256

# generating signing request for key, with a config file
openssl req -verbose -config ./openssl.cnf -new -key server.key -out server.csr -sha256 

# view CSR content
openssl req -in server.csr -noout -text

# using root CA to sign new key for server

## didnt work
touch ./demoCA/cacert.pem
openssl ca -out server-signed.crt -keyfile CA.key -verbose -infiles server.csr

## worked but didn't specify extended key usage or certificate revocation list
openssl x509 -req -config ./openssl.cnf -in server.csr -CA CA-signed.crt -CAkey CA.key -CAcreateserial -out server-signed.crt -addtrust serverAuth -days 9001 -sha256

# worked best
touch ./demoCA/cacert.pem
mkdir ./demoCA/private
touch ./demoCA/private/cakey.pem
openssl ca -in server.csr -out server-signed.crt -keyfile CA.key -cert CA-signed.crt -config ./openssl.cnf

# sometimes ran into an error when doing openssl ca, where nothing happened after password entry. removing index.txt fixed it.

# View contents of CRT:
openssl x509 -in server-signed.crt -text -noout

```

There are a lot of things going on in those lines, I'll try to explain them to the best of my understanding.

1. Generate a private key for your server to use to identify itself.  
1. Make a certificate sign request, with specific settings in a config file
1. Use the above two items to create a signed certificate for the server

The one tricky part is the config file settings, which are used in conjunction with the sign request and the actual signing of the server cert. I have included below the relevant sections of my conf file that I modified from the default version.



```cnf
#... initial code in config file 

# Section that is run during certificate requests
[ req ]

# ... A bunch of other stuff will be here

# The v3_req section will cover what to do for request extensions
req_extensions = v3_req

# ... other code ...

# v3_req section
[v3_req]

basicConstraints = CA:FALSE

# Key should only be used for these things. This should be important for Windows compatibility especially
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth, clientAuth

# The certificate revocation list (CRL) located at said URL will tell you if there are any bad certificates
crlDistributionPoints = URI:https://website.com/CA.crl

# refer to the section alt_names for other names for the CRL website
subjectAltName  = @alt_names

# alt_names custom section
[alt_names]
DNS.1 = website.com
DNS.2 = *.website.com

# other code ...

```
This conf file does a few things:

* Set default .csr field names so I can breeze through the organization/city/state settings when making the signing request (not shown)
* Specify usage for the signed certificate 
* Specify a certificate revocation list, where remote clients can check to see if this certificate has been compromised. If I don't specify this, I will get a warning message about it when I try to RDP from a remote.

I won't go into too much detail, mainly because I don't understand it well enough yet myself. Bottom line, now I have a neat, signed certificate that my server can broadcast, and because my remote clients trust my root CA, they can verify any certificate to ensure that is signed by my root CA. Now, all I need to do is to make my server broadcast the correct certificate.

## RDP certificate announcement

I've been calling it a server, but really I am just referring to my desktop computer. If I actually had windows server softare, there are several GUI-type tools provided by windows to generate certificates nicely for this. Since I don't have that, I had to do some custom key exporting and registry editing to make this work. I used [this](https://support.microsoft.com/en-gb/help/2001849/how-to-force-remote-desktop-services-on-windows-7-to-use-a-custom-serv) to help me figure it all out.

First, I needed to export my server private key and certificate out into a nice format for Windows. 

```bash
# create a pfx file from a private key and a certificate. It will ask you for a password.
openssl pkcs12 -export -out server.pfx -inkey server.key -in server-signed.crt 
```
Next, I need to double-click that .pfx file and import it into windows. I placed mine in the personal folder of the local computer certificates. You will need to use the same password that you just made during the pfx generation step. 

After importing, go to the certification manager (search "manage computer certificates" in the windows bar) and navigate to the certificate you imported. Right-click -> all tasks -> manage private keys, and add the user NETWORK SERVICE with READ permissions to the key. This will allow the computer's RDP to publish certs based on this key. While you are here, go double click on the server certificate in the cert manager and copy the "thumbprint" field.

Next, you have to make it so that your RDP will actually publish the cert that you want it to. I made a powershell script to do that 

```powershell
# paste your thumbprint here
$hex = "00112233445566778899AABBCCDDEEFF00112233"

# make uppercase and put commas between every two characters except the last pair
$hex = $hex.toUpper() -replace '(..(?!$))','$1,'

# split/add newlines on the commas you just made (probably could be optimized), and prefix with 0x to denote hex
$hexified = $hex.Split(',') | % { "0x$_"}

# set the following registry key with the name SSLCertificateSHA1Hash and with the value as the hex values cast to byte format
New-ItemProperty -Force -Path "HKLM:\SYSTEM\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp\" -Name "SSLCertificateSHA1Hash" -PropertyType Binary  -Value ([byte[]]$hexified)
```

That was the final step! Now, restart your computer/restart the RDP service, and when you try to remote in to this computer, you should not see any warnings!


... not going to lie, you will probably still get warnings. 

## Certificate revocation lists

If you were paying attention, you saw that I referenced CRLs, and that there were some lines in the openssl configuration file listed above about CRL URLs. The CRL is like a wanted list of misbehaving certificates. Most computers are set up to check any given CRLs to make sure it's not gettting a bad certificate. If for some reason a certificate doesn't give up a reference CRL, the computer will get suspicious and warn the user about it. That's what happened to me.. so I made a certificate revocation list and uploaded it to my website, mostly following instructions from [this](https://blog.didierstevens.com/2013/05/08/howto-make-your-own-cert-and-revocation-list-with-openssl/). 

```bash
# Generating certificate revocation list file for root CA
echo 100 > ./demoCA/crlnumber
openssl ca -config ./openssl.cnf -gencrl -keyfile CA.key -cert CA-signed.crt -out CA.crl.pem
openssl crl -inform PEM -in CA.crl.pem -outform DER -out CA.crl
```

I then uploaded the CA.crl file to my website (it would have to be located at website.com/CA.crl as specified in the config file, and the warning went away as expected.

## Other things to note

### Passwords

A lot of the steps had passwords involved. I am not sure which ones needed to be set, so I made a new password whenever it asked me for one. Don't forget those passwords!

### Root CA security

This entire system falls apart if your root CA is compromised! keep it somewhere safe and disconnected from the internets!!

### Name consistency

In order for things to get verified correctly, fields related to the company name, user name, and location must be set the same between root and subordinate CAs!














