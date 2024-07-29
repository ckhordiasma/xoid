---
layout: post
title: "Encrypting and Decrypting Email"

---

At my current workplace, emails are frequently frequently encrypted/decrypted in Microsoft Outlook, using a smart card that is issued to every employee. At work, we primarily use Windows, and either the default Microsoft authentication tools or middleware such as ActivClient are used in order to interface with our smart cards / smart card readers. Until I started this project yesterday, I had only ever used GUI applications in order to do anything related to my smartcard. 

Yesterday, I set out with a goal of learning how to encrypt/decrypt things with my smartcard in linux, with an end goal of figuring out how to decrypt email messages from work, and an ultimate goal of building my own application to do this task. I was driven to learn this after several unsuccessful attempts to use my work's version of webmail during business travel.. our webmail is supposed to have the ability to open encrypted emails, but more often than not, that feature does not work (and it requires internet explorer, of all things)! In webmail, instead of showing the message, an "smime.p7m" encrypted attachment is given. So my goal was to be able to freely decrypt those smime messages using my smartcard from my home linux computer.

So, first things first: [googling around / finding tutorials](https://github.com/OpenSC/OpenSC/wiki/Quick-Start-with-OpenSC) and installing a bunch of stuff.  

```bash
sudo apt-get install openssl opensc pcsc_tools
```

`openssl` is a general-purpose toolset used to encrypt stuff. It has a lot of options and different configurations.
`opensc` is a package used for interfacing with a smart card.
I'm honestly not sure what `pcsc_tools` is but I installed it anyways because some tutorials had it.

Once I got those installed, I did some poking around:

```bash
# check to make sure a smart card reader was there
opensc-tool --list-readers

#checks on reader#0, this shows some hex identifying the card. ATR = answer to reset
opensc-tool --reader 0 --atr

#this'll tell you if the card is supported or not. When i tried a normal credit card, it said "Unsupported Card". When I tried my smartcard, it said "PIV-II Card"
opensc-tool --reader 0 --name

#this is supposed to be able to look around your smart card. I tried it with my smartcard and got an error: "Unable to select MF: file not found"
opensc-explorer
```

This is mostly following the tutorial linked above. I skipped the smartcard setup steps because my smartcard from work is already configured. The next line I tried was straight from the tutorial but did not work:

```bash
openssl
engine dynamic -pre SO_PATH:/usr/lib/engines/engine_pkcs11.so -pre ID:pkcs11 -pre LIST_ADD:1 -pre LOAD -pre MODULE_PATH:opensc-pkcs11.so
```

This did not work, saying it coukld not find the SO_PATH. Rereading the tutorial, I determined that I needed to download this [package](https://github.com/OpenSC/libp11/blob/master/INSTALL.md) and install it. Installing it was easy, I just followed the install instructions on the website (untarball it and run the config/make/install commands). 

During my process of figuring things out, I also installed some packages `sudo apt-get install libp11-3 libengine-pkcs11-openssl` but I'm honestly not sure if this was even necessary or redundant. I am including it here for completeness.

After all that, I was able to run the modified command (note difference in SO_PATH as well), without errors.

```bash
openssl
engine dynamic -pre SO_PATH:/usr/lib/x86_64-linux-gnu/engines-1.1/pkcs11.so -pre ID:pkcs11 -pre LIST_ADD:1 -pre LOAD -pre MODULE_PATH:opensc-pkcs11.so
```

By without errors, I mean that it gave me back a bunch of SUCCESS bars without any ERROR bars.

So great! I was able to show that I could a) read smart cards at a low level and b) do some basics on interfacing the card with openssl. Now, how do I actually encrypt or decrypt things with my smartcard??

The first two things I tried did not work:

```bash 
#This is to create a new certificate request, did not work
req -engine pkcs11 -new -key id_45 -keyform engine -x509 -out cert.pem -text

#looked around in other websites while googling, saw this, got to the point where it asked me for my pin, but pin did not seem to work!
pkcs15-crypt -s -i ./text.txt -k 2 -o ./outfile.txt -v -w
```

That made me step back and think, why am I trying a pkcs15 thingy when I used openssl before with a pkcs11 thingy? I clearly should be using something pkcs11 related:

```bash
#smartcard login test
pkcs11-tool -l -t
#view all things on smartcard
pkcs11-tool -O 
```

That did the trick! I was able to type my PIN into the card and see stuff in the card! Looking at the help page for pkcs11-tool, it seems like I can use it for encryption:

```bash
#After trying a few things, I was able to sign something with 
pkcs11-tool -v -s -m RSA-X-509 -i text.txt -o outfile.txt
#but when i tried to decrypt it, it wasn't correct, AT ALL! 
pkcs11-tool -v --decrypt -m RSA-X-509 -i outfile.txt -o backinfile.txt 
```

Clearly, I was using this tool incorrectly. Since I was stuck, I went on google some more and found [this other article](https://github.com/OpenSC/OpenSC/wiki/Using-pkcs11-tool-and-OpenSSL) from the OpenSC wiki, and in the second half it talks about encrypting stuff with a smartcard. Just what I wanted!

Encryption is a confusing process, and it gets more confusing the more "foolproof" your encryption method is. I'm not going to try very hard to explain it, because I am not an expert in it myself. 

The first thing you need to do is get your public key from the smartcard, using `pkcs11-tool`. You can convert and use this key with `openssl` commands to encrypt data, and then later when you want to decrypt the data, you can ask your smart card nicely (i.e. type the correct PIN) to decrypt the data for you, using the private key stored on the card via `pkcs11-tool` commands.

```bash
#this saves the public encryption cert, as key.cert
pkcs11-tool -r --id 2 --type cert > key.cert

#this converts the cert into the PEM/PUB format
openssl x509 -inform DER -in key.cert -pubkey > key.pub
```

Now, time to make some data to encrypt:
```bash
echo "hello world! hello world again! I need to make this file big enough or else it won't encrypt properly! hello world! hello world! hello worl" > data
```

The website gave a few different methods of encryption. For the RSA-X-509 method, some data padding was needed before encryption.

```bash
#RSA-X-509 standard requires some kind of padding before encryption
(echo -ne "\x00\x02" && for i in `seq 113`; do echo -ne "\xff"; done && echo -ne "\00" && cat data) > data_pad

# then encrypt with public key that you made earlier
openssl rsautl -encrypt -inkey key.pub -in data_pad -pubin -out data_pad.crypt -raw
```

You'll notice that my data string is wierd and truncated. I had tried just "hello world!" in the beginning, but I got errors for it being too short. Then I tried something significantly longer, but then I got errors for it being too long! after some guesswork I was able to find the goldilocks size to get stuff encrypted. If I had read the encryption standards, it would have probably told me about these specific length requirements.

Once I was able to encrypt something, I could decrypt it with the command

```bash
pkcs11-tool --id 2 --decrypt -m RSA-X-509 --input-file data_pad.crypt -o data_pad_decrypted
```

Great! when I opened up `data_pad_decrypted`, it showed up as the plaintext I was expecting. Great, I was able to encrypt some short piece of text... maybe great for authentication purposes, but not that great for encrypting emails. How the heck to I encrypt larger things? My first thought was to use one of the other encryption methods listed in the tutorial.

```bash
#how do I encrypt bigger things?? maybe I need to use a RSA-PKCS instead
echo "hello world! hello world again! I need to make this file big enough or else it won't encrypt properly! hello world! hello world! hello world! hello world! hello world again! I need to make this file big enough or else it won't encrypt properly! hello world! hello world! hello world!" > data2

#the below didn't work, said it was too big
openssl rsautl -encrypt -inkey key.pub -in data2 -pubin -out data2.crypt
```

All right, clearly I am stuck and need help from the internet. I found [this](https://stackoverflow.com/questions/7143514/how-to-encrypt-a-large-file-in-openssl-using-public-key), which told me to use the smime type of openssl! Wow, that's the same name as the thing that my encrypted emails spit out! I bet they are related!

Note that the encryption terminology is almost the same as earlier encryption commands. I'm still using the public key off the smartcard found using `pkcs11-tool` and it's still using the `openssl` command.

```bash
#this was able to encrypt large data! finally!
openssl smime -encrypt -binary -aes-256-cbc -in data2 -out data2.crypt -outform DER key.pub
```

Decrypting ended up being a little more tricky. I couldn't get pkcs11-tool to figure out how to decrypt the smime encryption type, which I attempted using:

```bash
pkcs11-tool --id 2 --decrypt --input-file data2.crypt -o data2_decrypted
```

Doing some more [googling around](https://gist.github.com/kpe/15dcfc7ed46321347320faa65eacfb7d), I discovered something very fundamental and important: Instead of using `pkcs11-tool`, I could use `openssl` commands with a customized openssl configuration file and decrypt things that way!

Contents of configuration file, `pkcs11_engine.conf`

```
openssl_conf = openssl_init

[openssl_init]
engines = engine_section

[engine_section]
pkcs11 = pkcs11_section

[pkcs11_section]
engine_id = pkcs11
dynamic_path = /usr/lib/x86_64-linux-gnu/engines-1.1/pkcs11.so
MODULE_PATH = /usr/lib/x86_64-linux-gnu/opensc-pkcs11.so
init = 0
```

Actual decryption command:

```bash
export OPENSSL_CONF=./pkcs11_engine.conf 
openssl smime -decrypt -inform der -in data2.crypt -engine pkcs11 -keyform engine -inkey 2 > data2_decrypted
```

THAT WORKED!!!!! I got the data back exactly the same as before I encrypted it. Now I need to figure out how to use it to open those pesky `smime.p7m` files.

I had no idea what certificate in the smart card I was supposed to use to decrypt the message. Luckily, the smartcard only had three keys, and after some guesswork, I determined that I needed to use the 3rd key. This is what I put into the `-inkey` parameter.

```bash
#trying on some actual data:
export OPENSSL_CONF=./pkcs11_engine.conf 
openssl smime -decrypt -inform der -in smime.p7m -engine pkcs11 -keyform engine -inkey 3 > email_msg
```

I also tried some other variants:
```bash
#this one didn't work (text option)
openssl smime -decrypt -inform der -in smime.p7m -engine pkcs11 -keyform engine -inkey 3 -text > email_msg_noheader

#this one worked, not sure if it is any better (binary option)
openssl smime -decrypt -inform der -in smime.p7m -engine pkcs11 -keyform engine -inkey 3 -binary > email_msg

#also tried this, not sure if any better (binary and noverify option)
openssl smime -decrypt -inform der -in smime.p7m -engine pkcs11 -keyform engine -inkey 3 -binary -noverify > email_msg
```

Using some of the above commands, I was able to decrypt the message! but with a catch.. From looking at the raw file, it was clear that the message was still base64 encoded pretty hard. openssl handily has a base64 decoding feature, which I used as below:

```bash
openssl base64 -d -in email_msg -out message.email
```

Finally, I was able to get some human-readable output! However, `message.email` wasn't a nicely formed text file; it had a lot of strange binary at the beginning, and then the file was broken up into several sections: a plaintext email section, HTML email section, and subsequent sections corresponding to attachments. The sections were delinated by a some text `------=_NextPart_000_0027_01D525BE.AB6C19A0` (this is defined at the beginning of the file). In typical inception fashion, the attachment sections were further base64-encoded binary data (my test email had a word and pdf document).

After all this work, I was able to decrypt data, but did not quite get to the point where I could extract all usable data. I could read message body text fine, but I need to do more work to get the attachemnts extracted out.




















