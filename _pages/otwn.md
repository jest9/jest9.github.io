---
layout: post
title: "OverTheWire Natas"
date: 2024-10-04 10:04:00 +0000
categories: Writeup
---

Answer to every <span style="color:blue">natas</span> step.

General descriptions.

&nbsp;

# Level 0

Password for next page is located inside the HTML page, acquired via inspect element.

&nbsp;

# Level 1

Same thing but right click is blocked, use ctrl+shift+i hotkey.

&nbsp;

# Level 2

Inspect element to find png which shows a /files directory exists, which contains a users.txt file containing the credentials for natas3.

&nbsp;

# Level 3

Viewing robots.txt reveals a secret directory anmed s3cr3t, containg a users.txt file and natas4 credentials.

&nbsp;

# Level 4

Page displays that only users from natas5 can see the page contents. Opening burpsuite and making the request I can add the referer header with the value of natas5 to get the page to display and give the password for natas5.

&nbsp;

# Level 5

Page request contains a value cookie set to 0 to verify login status. By switching this to 1 and making the request we gain access to the credentials for natas6.

&nbsp;

# Level 6

Page contains basic secret mechanism, where the source code exposes that the secret is being pulled from a file at /includes/secret.inc, which is accessible. Using that we get natas7 credentials.

&nbsp;

# Level 7

Basic ui where index.php uses the page parameter to dynamically determine what content to load on index.php. It is vulnerable to LFI, such that index.php?page=../../../etc/passwd displays the /etc/passwd system file. The password is located in the /etc/natas_webpass/natas8 file.

&nbsp;

# Level 8

Requires a secret to get credentials. The source code exposes the encoded secret with the proces used to encode it. We reverse the encoded secret by decoding it from hex and then reversing the string, followed by decoding from base64. Upon entering this we get credentials for natas9.

```php
$encodedSecret = "3d3d516343746d4d6d6c315669563362";

function encodeSecret($secret) {
    return bin2hex(strrev(base64_encode($secret)));
}
```

&nbsp;

# Level 9

Just a basic dictionary, searches for string given in dictionary file. As the underlying mechanism uses grep on user input ($key) it allows for command injection. By visiting http://natas9.natas.labs.overthewire.org/?needle=;cat%20/etc/passwd, the semi-colon causes it to end the command and displays the /etc/passwd file.

Visiting /etc/natas_webpass/natas10 provides us with the natas10 password.

```php
<?
$key = "";

if(array_key_exists("needle", $_REQUEST)) {
    $key = $_REQUEST["needle"];
}

if($key != "") {
    passthru("grep -i $key dictionary.txt");
}
?>
```

&nbsp;

# Level 10

Same thing as before however with filtered characters (except `, however substitution didn't appear to work.). We can alter the grep query by entering "a /etc/passwd", exploiting the fact that grep can search through multiple files at once. We can use this to get the password from /etc/natas_webpass/natas11.

&nbsp;

# Level 11
