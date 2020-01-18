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
  * FTP Banner Grabbing
  * Source Code Analysis (Web)
  
  
# 0x00 Quick Summary
hackNos: ReconForce was a quite easy box, took me around 1day to root it because i was using the broken version. The mistake takes place at FTP Banner, it should be `Security@hackNos` & not `Secure@hackNos`. I wasted lot of time brute forcing the HTTP Authentication, when i got the patched version i was able to root it into 10 minutes. FTP Banner provide us the HTTP Authentication password, when we login in we can see a ping function, we need to bypass it & take command injection. Then we have to do 2 privilege escalations, very easy ones.

# 0x01 Enumeration
## Finding Target's IP Address

  
