---
layout: post
title: "Instant"
date: 2024-10-16 10:37:00 +0000
categories: Writeup
---
Instant Writeup by <span style="color:yellow">jest</span>.

# Scan

```
Nmap scan report for 10.129.42.207
Host is up (0.026s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 31:83:eb:9f:15:f8:40:a5:04:9c:cb:3f:f6:ec:49:76 (ECDSA)
|_  256 6f:66:03:47:0e:8a:e0:03:97:67:5b:41:cf:e2:c7:c7 (ED25519)
80/tcp open  http    Apache httpd 2.4.58
|_http-title: Did not follow redirect to http://instant.htb/
|_http-server-header: Apache/2.4.58 (Ubuntu)
Service Info: Host: instant.htb; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 29.91 seconds
```

The site has an APK to install, which I then decompiled using jadx:

![image](https://github.com/user-attachments/assets/c5fd4760-0da2-4538-9441-788d08dfce4e)

From viewing this file I have now a JWT:

![image](https://github.com/user-attachments/assets/7effe06e-6dda-4241-bc78-fcd2e58a9288)

From further inspection we also find a subdomain:

![image](https://github.com/user-attachments/assets/84ca14a7-1440-4775-99a3-7f45f97cc490)

This could be where we can use the token we got, or something.
