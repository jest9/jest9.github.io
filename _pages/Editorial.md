## Initial Enumeration

![eadbd8ba866fc556a9b52caa9c0e8075.png](../../../_resources/eadbd8ba866fc556a9b52caa9c0e8075.png)

Starting off with nmap scan of the address:

```text
Nmap scan report for 10.10.11.20
Host is up (0.0077s latency).
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.7 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   256 0d:ed:b2:9c:e2:53:fb:d4:c8:c1:19:6e:75:80:d8:64 (ECDSA)
|_  256 0f:b9:a7:51:0e:00:d5:7b:5b:7c:5f:bf:2b:ed:53:a0 (ED25519)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
|_http-title: Did not follow redirect to http://editorial.htb
|_http-server-header: nginx/1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.39 seconds
```

We have a domain to throw into /etc/hosts, which I do, we can also consider subdomain fuzzing.

```bash
wfuzz -H "Host: FUZZ.editorial.htb" -w /usr/share/SecLists/Discovery/DNS/subdomains-top1million-110000.txt --hh 178 http://editorial.htb
```

No results.

&nbsp;

&nbsp;

## Website Enumeration

```text
##_|. _ _  _  _  _ _|_    v0.4.3
 (_||| _) (/_(_|| (_| )

Extensions: php, aspx, jsp, html, js | HTTP method: GET | Threads: 25 | Wordlist size: 11460

Output File: /home/jest/reports/http_editorial.htb/_24-06-25_15-05-36.txt

Target: http://editorial.htb/

[15:05:36] Starting:
[15:05:42] 200 -    3KB - /about
[15:06:27] 200 -    7KB - /upload

Task Completed
```

There's an upload, so potential for a reverse shell or webshell which we can use to execute commands.

![387552ac6c5ae91196ec4d38adb64cd4.png](../../../_resources/387552ac6c5ae91196ec4d38adb64cd4.png)

Another site? enter into /etc/hosts and try this out.

Same place, no find.

&nbsp;

&nbsp;

## SSRF

We can upload jpeg files:

![38a46a88d27743eb37339fcd77c99567.png](../../../_resources/38a46a88d27743eb37339fcd77c99567.png)

We can try and trick the server into thinking we are uploading a jpeg file via altering the hexcode.

You have to use burpsuite to identify that via preview you can can SSRF by putting the the localhost address in the cover url and sending via preview (which I found.).

What I didn't find was the idea that you are to use this to enumerate a port open locally on 5000, which you then can interact with by making requests and DOWNLOADING the file at the link it returns to get the output, which reveals a user and password, complete bullshit.

![560105043f9e861b7c630c07de02151c.png](../../../_resources/560105043f9e861b7c630c07de02151c.png)

![5680857e8f95013d5c6f646d2fe0517c.png](../../../_resources/5680857e8f95013d5c6f646d2fe0517c.png)

```bash
wget http://editorial.htb/static/uploads/386cb67e-76d4-4cd9-b6b8-7d9d81ccba5f
```

![23b1927b5b98480bcb09ba52a09fb2f5.png](../../../_resources/23b1927b5b98480bcb09ba52a09fb2f5.png)

We can use these with burp to make endpoint api queries:![8f178b7a3fcb8adbbd151fc07b84346e.png](../../../_resources/8f178b7a3fcb8adbbd151fc07b84346e.png)![fdb661dddde3babdf46016ae8e32f41d.png](../../../_resources/fdb661dddde3babdf46016ae8e32f41d.png)![9b5f30155cf52ba9490dfb89ee511d28.png](../../../_resources/9b5f30155cf52ba9490dfb89ee511d28.png)

It's ssh credentials.

&nbsp;

&nbsp;

&nbsp;

## Privesc

## Shell as Prod

![a4caa57bf9ed4341178552bedfa67895.png](../../../_resources/a4caa57bf9ed4341178552bedfa67895.png)

Doesn't look too promising in terms of what we can get running as root, clear.sh is owned and ran as www-data sadly.

There's another user named prod, and this is what they own (from what we can see as dev.):

![116f87fccad5f5aca9f804f6adcc8dee.png](../../../_resources/116f87fccad5f5aca9f804f6adcc8dee.png)

There's a .git directory in dev home:

![4fb4d230a6852e89c2c7ff0e5783c08e.png](../../../_resources/4fb4d230a6852e89c2c7ff0e5783c08e.png)

Maybe we can find something in commit messages or something.

![6f95f1fd8dda65e950a75ed5fd697f9c.png](../../../_resources/6f95f1fd8dda65e950a75ed5fd697f9c.png)

commit b73481bb823d2dfb49c44f4c1e6a7e11912ed8ae seems interesting due to the mention of prod.

![da4ac6994fb8a5570c82d9bb1f1334c3.png](../../../_resources/da4ac6994fb8a5570c82d9bb1f1334c3.png)

We now have our prod creds.

080217_Producti0n_2023!@

&nbsp;

&nbsp;

## Shell as Root

![348cd62e5ffd717b6debb56a6521d37e.png](../../../_resources/348cd62e5ffd717b6debb56a6521d37e.png)

We must inspect the binary, but there must be something we can do here, maybe wildcard expansion related.

I thought that perhaps I could clone the root flag or something, but had no idea where to start when doing that.

```python
#!/usr/bin/python3

import os
import sys
from git import Repo

os.chdir('/opt/internal_apps/clone_changes')

url_to_clone = sys.argv[1]

r = Repo.init('', bare=True)
r.clone_from(url_to_clone, 'new_changes', multi_options=["-c protocol.ext.allow=always"])
```

We can see it run on whatever directory we go to, there is no limitation:

```bash
stderr: 'fatal: repository 'test' does not exist
```

&nbsp;

GTFOBins suggests we could try pass arguments in quotation marks, which seems to work very well:![0dcd37acf8c54625feb32a2452d09b6c.png](../../../_resources/0dcd37acf8c54625feb32a2452d09b6c.png)

So now we just need to find a way of exploiting this.

So what I was missing was this:

```python
-c protocol.ext.allow=always
```

This is external protocols, which are ways git can run an external command as if it was a URL.

So I was trying to find it manually, but there's a post if you just google python3 git exploit:

https://github.com/gitpython-developers/GitPython/issues/1515

Doesn't seem to be working due to the new_changes directory having an unremovable .git repo there.

Judging by the writeups the box is...

&nbsp;

# BROKEN.

&nbsp;

Since HTB let's multiple users onto boxes, if someone were to exploit this and write something to the new_changes directory (as it would with the exploit), it bricks entirely, and as that which is written to new_changes is written as ROOT, we can't do anything about it. Nice.