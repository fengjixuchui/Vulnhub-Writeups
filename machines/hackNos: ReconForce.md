# Vulnhub | hackNos: ReconForce Writeup (In Detail)
## [+] Machine Details

| INFO | Description |
| --- | --- |
| [+] Machine Name -> | ``hackNos: ReconForce`` |
| [+] Release Date -> | ``11 Jan 2020`` |
| [+] Difficulty -> | ``Easy/Medium`` |
| [+] Download -> | [Download Link](https://www.vulnhub.com/entry/hacknos-reconforce,416/) |
| [+] Writeup By -> | ``@pwn4magic`` - `Discord : pwn4magic#8707` |
| [+] VM Maker -> | [@rahul_gehlaut](https://twitter.com/rahul_gehlaut) |
| [+] OS -> | ``GNU/Linux`` :penguin: |


## [+] Table of contents
* 0x00 Quick Summary
* 0x01 Enumeration
  * Finding Target's IP Address
  * nmap
  * FTP Enumeration / FTP Banner Grabbing
  * Web Enumeration
  * Theory (.htaccess/.htpasswd)
  
  
# 0x00 Quick Summary
hackNos: ReconForce was a quite easy box, took me around 1day to root it because i was using the broken version. The mistake takes place at FTP Banner, it should be `Security@hackNos` & not `Secure@hackNos`. I wasted lot of time brute forcing the HTTP Authentication, when i got the patched version i was able to root it into 10 minutes. FTP Banner provide us the HTTP Authentication password, when we login in we can see a ping function, we need to bypass it & take command injection. Then we have to do 2 privilege escalations, very easy ones.

# 0x01 Enumeration
## Finding Target's IP Address
Let's find target's IP address, i always use `arp-scan`.
```
[root@pwn4magic]:~# arp-scan --localnet | grep "PCS Systemtechnik GmbH"
192.168.1.2	08:00:27:0c:93:5f	PCS Systemtechnik GmbH
```

## nmap
```
[root@pwn4magic]:~/Documents/vulnhub/hackNosReconForce# nmap -sC -sV -p- -oN nmap 192.168.1.2
Starting Nmap 7.80 ( https://nmap.org ) at 2020-01-18 16:18 EET
Nmap scan report for thepurge (192.168.1.2)
Host is up (0.00020s latency).
Not shown: 65532 closed ports
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 2.0.8 or later
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to ::ffff:192.168.1.14
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      At session startup, client count was 1
|      vsFTPd 3.0.3 - secure, fast, stable
|_End of status
22/tcp open  ssh     OpenSSH 8.0p1 Ubuntu 6build1 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 6f:96:94:65:72:80:08:93:23:90:20:bc:76:df:b8:ec (RSA)
|   256 6f:bb:49:1a:a9:b6:e5:00:84:19:a0:e4:2b:c4:57:c4 (ECDSA)
|_  256 ce:3d:94:05:f4:a6:82:c4:7f:3f:ba:37:1d:f6:23:b0 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title:  Recon_Web
MAC Address: 08:00:27:0C:93:5F (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

We can see, 3 open ports (21, 22, 80). Let's start by enumerating FTP.

## FTP Enumeration / FTP Banner Grabbing
  
FTP Anonymous is allowed, let's try it out.
```
[root@pwn4magic]:~/Documents/vulnhub/hackNosReconForce# ftp 192.168.1.2
Connected to 192.168.1.2.
220 "Security@hackNos".
Name (192.168.1.2:root): anonymous
331 Please specify the password.
Password:
230 Login successful.
Remote system type is UNIX.
Using binary mode to transfer files.
ftp> ls -la
200 PORT command successful. Consider using PASV.
150 Here comes the directory listing.
drwxr-xr-x    2 0        117          4096 Jan 06 06:25 .
drwxr-xr-x    2 0        117          4096 Jan 06 06:25 ..
226 Directory send OK.
ftp> 
```

Nothing useful there. But we can see at FTP banner this `Security@hackNos`, might be useful later. We can also do banner grabbing using netcat!

```
[root@pwn4magic]:~/Documents/vulnhub/hackNosReconForce# nc -v 192.168.1.2 21
thepurge [192.168.1.2] 21 (ftp) open
220 "Security@hackNos".
```

## Web Enumeration

Site doesnt have something really useful, i even tried stego on the photos. But if we check the source code we can see an interesting path. `/5ecure`

```html
<div class="button">
<a href="5ecure/" class="btn">TroubleShoot</a>>
</div>
```

![httpauth](https://i.imgur.com/KvMcjF8.png)

HTTP Authentication, let's see some background theory.

## Theory (.htaccess/.htpasswd)
