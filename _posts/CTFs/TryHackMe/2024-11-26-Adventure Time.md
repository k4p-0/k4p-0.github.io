---
title: Adventure Time
date: 2024-11-26
categories: [CTF, TryHackMe]
tags: [Steganography, Encryption, Linux]
description: A walkthrough of the Adventure Time room
---

[Adventure Time Room](https://tryhackme.com/r/room/adventuretime)

## Enumeration

Nmap Scan:

```terminal
┌──(kali㉿kali)-[~/Adventure]
└─$ sudo nmap -sS -p- -T4 10.10.70.125 -oN TCP.nmap        
[sudo] password for kali: 
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-11-14 13:20 EST
Nmap scan report for 10.10.70.125
Host is up (0.095s latency).
Not shown: 65530 closed tcp ports (reset)
PORT      STATE SERVICE
21/tcp    open  ftp
22/tcp    open  ssh
80/tcp    open  http
443/tcp   open  https
31337/tcp open  Elite

Nmap done: 1 IP address (1 host up) scanned in 914.61 seconds
```

## Flag 1

Looking through the webpages we find this page on `https://ip_address/`

![Web Page Clue](/assets/img/CTFs/Adventure_Time/https_page_1.jpg "test")

Using a Gobuster scan we find another directory called `/candybar/`

![Gobuster Scan](/assets/img/CTFs/Adventure_Time/gobuster_1.jpg)

This leads us to a webpage that has a clue

![Web Page Clue](/assets/img/CTFs/Adventure_Time/https_page_2.jpg)

Using [CyberChef](https://cyberchef.org/) and [dCode](https://www.dcode.fr/en) we are able to decrypt this clue using base32 and a ROT cipher

![Base32](/assets/img/CTFs/Adventure_Time/cyberchef_1.jpg)

![ROT Cipher](/assets/img/CTFs/Adventure_Time/dcode_1.jpg)

Checking the SSL Certificate gives us a domain name to use

![SSL Certificate](/assets/img/CTFs/Adventure_Time/ssl_cert.jpg)

Editing the `/etc/hosts` file allows us the connect the the IP address with `adventure-time.com`

![/etc/hosts](/assets/img/CTFs/Adventure_Time/host_1.jpg)

Unfortunately that does not change anything about the webpages, however the SSL certificate does have another domain name `land-of-ooo.com`

![/etc/hosts](/assets/img/CTFs/Adventure_Time/host_2.jpg)

Changing the `/etc/hosts` file to `land-of-ooo.com` lets us find a new webpage at `https://land-of-ooo.com`

![Web Page](/assets/img/CTFs/Adventure_Time/land_page_1.jpg)

Using Feroxbuster to do a recursive directory brute force, finds us three new directories.  

![feroxbuster](/assets/img/CTFs/Adventure_Time/ferox_1.jpg)

The `/yellowdog/` only gives us a bit of information 

![yellowdog](/assets/img/CTFs/Adventure_Time/yellowdog.jpg)

The `bananastock` directory gives us the banana guards' password, however it is encrypted in morse code

![bananastock](/assets/img/CTFs/Adventure_Time/bananastock.jpg)

After decrypting the message we get the banana guards' password

![Morse Code](/assets/img/CTFs/Adventure_Time/cyberchef_2.jpg)

The `/princess/` page informs us that the banana guards' username has been changed and that the new username is in a "secret safe"

![princess](/assets/img/CTFs/Adventure_Time/princess.jpg)

investigating the page further reveals encrypted text in the source code 

![Source Code](/assets/img/CTFs/Adventure_Time/source.jpg) 

We can use AES decryption to find the meaning of the text

![AES Decrypt](/assets/img/CTFs/Adventure_Time/cyberchef_3.jpg)

Accessing the port at `31337` and supplying the word `ricardio` reveals the new name of the banana guards

![Telnet](/assets/img/CTFs/Adventure_Time/telnet_1.jpg)

Using these credentials we can use ssh to login to the banana guards' account and find the first flag

![SSH](/assets/img/CTFs/Adventure_Time/ssh_1.jpg)

![flag1](/assets/img/CTFs/Adventure_Time/flag_1.jpg)

## Flag 2

In the apple-guards' home directory we find the `mbox` file that contains a message from marceline, that states they hid a file that will help improve our access

![mbox](/assets/img/CTFs/Adventure_Time/mbox_1.jpg)

We can find that file by searching the filesystem for files that are owned by the user marceline

![find](/assets/img/CTFs/Adventure_Time/find_1.jpg)

Running the file `helper` we are greeted with another clue, and we are given the encrypted text `Gpnhkse` along with a key `gone`

![helper](/assets/img/CTFs/Adventure_Time/helper_1.jpg)

Using a Vigenere Cipher we are able to decrypt the text

![Vigenere Cipher](/assets/img/CTFs/Adventure_Time/vigenere_1.jpg)

Using the answer, we are given the password to marceline's account

![helper](/assets/img/CTFs/Adventure_Time/helper_2.jpg)

We can switch to marceline's account and find the flag in their home directory

![marceline](/assets/img/CTFs/Adventure_Time/marceline.jpg)

![Flag 2](/assets/img/CTFs/Adventure_Time/flag_2.jpg)

## Flag 3

In marceline's home directory the file `I-got-a-secret.txt` contains another clue

![I-got-a-secret.txt](/assets/img/CTFs/Adventure_Time/secret.jpg)

The binary code can be decrypted using a spoon interpreter

![spoon](/assets/img/CTFs/Adventure_Time/spoon.jpg)

Using the new magic word `ApplePie` at port `31337` gives us the password for peppermint-butler

![telnet](/assets/img/CTFs/Adventure_Time/telnet_2.jpg)

Switching to the peppermint-butler account, you can find the flag in their home directory

![Flag 3](/assets/img/CTFs/Adventure_Time/flag_3.jpg)

## Flag 4

In peppermint-butler's home directory we can find a file called `butler-1.jpg`, we can copy that so we can take a closer look at it

![scp](/assets/img/CTFs/Adventure_Time/scp.jpg)

However, upon initial inspection nothing immediately stands out about the image

![butler](/assets/img/CTFs/Adventure_Time/butler.jpg)

Continuing our enumeration, we find two interesting files when we search the filesystem for files owned by the user peppermint-butler

![find](/assets/img/CTFs/Adventure_Time/find_2.jpg)

![find](/assets/img/CTFs/Adventure_Time/find_3.jpg)

These contain passwords that will allow us to find hidden information inside `butler-1.jpg`

![steg.txt](/assets/img/CTFs/Adventure_Time/steg.jpg)

![zip.txt](/assets/img/CTFs/Adventure_Time/zip.jpg)

These passwords allow us to extract secret data hidden inside the image, and find most of gunter's password

![Extract Data](/assets/img/CTFs/Adventure_Time/extract.jpg)

Using this we are able to guess the rest of the password, gain access to gunter's account, and get the flag.

![Flag 4](/assets/img/CTFs/Adventure_Time/flag_4.jpg)

If you are unable to guess the password you could use crunch(`crunch 18 18 -t "The Ice King s@@@@" -o passwords.txt`) to generate a wordlist, that will allow you to gain access through ssh via a brute force attack

## Flag 5

While we are enumerating the machine, the `exim4` binary stands out when we are checking for SUID vulnerabilities 

![SUID](/assets/img/CTFs/Adventure_Time/suid.jpg)

Upon further investigation this version of exim4 seems to be exploitable

![dpkg](/assets/img/CTFs/Adventure_Time/dpkg.jpg)

![searchsploit](/assets/img/CTFs/Adventure_Time/searchsploit_1.jpg)

After finding our exploit the next step is to get the exploit onto the target machine

![searchsploit copy](/assets/img/CTFs/Adventure_Time/searchsploit_2.jpg)

![python server](/assets/img/CTFs/Adventure_Time/python_server.jpg)

![wget](/assets/img/CTFs/Adventure_Time/wget.jpg)

To get the exploit to work we need to make sure that we are connecting to the correct port, we can check the port the service is running on in `/etc/exim4/update-exim4.conf.conf` 

![exim conf](/assets/img/CTFs/Adventure_Time/conf.jpg)

Now we need to edit the exploit to make it target the correct port

![nano](/assets/img/CTFs/Adventure_Time/nano.jpg)

![Edit Exploit](/assets/img/CTFs/Adventure_Time/edit_exploit.jpg)

Finally, we get run the exploit and get root

![exploit](/assets/img/CTFs/Adventure_Time/exploit.jpg)

now we can get the final flag, which is located in `/home/bubblegum/secrets`

![Secret Directory](/assets/img/CTFs/Adventure_Time/secret_dir.jpg)

![Flag 5](/assets/img/CTFs/Adventure_Time/flag_5.jpg)
