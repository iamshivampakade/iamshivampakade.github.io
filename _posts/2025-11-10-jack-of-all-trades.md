---
title: "Jack-of-all-trades — TryHackMe Walkthrough"
date: 2025-11-06
author: shivam
categories: [ctf, tryhackme, writeup]
tags: [jack-of-all-trades, TryHackMe, RCE, privilege-escalation]
summary: "Full walkthrough for the TryHackMe 'jack-of-all-trades' room — recon, web RCE, credential harvesting, shell, and privilege escalation."
image:
  path: /assets/img/TryHackMe/jacktrades/jacktrades.jpeg
  alt: "Jack-of-all-trades — site header"
---

## Table of contents

1. [Recon and port scan](#recon-and-port-scan)
2. [Web discovery and quick probing](#web-discovery-and-quick-probing)
3. [Remote command execution (RCE) via cmd parameter](#remote-command-execution-rce-via-cmd-parameter)
4. [Harvesting jacks_password_list and decoding](#harvesting-jacks_password_list-and-decoding)
5. [Initial access — get a shell as jack](#initial-access--get-a-shell-as-jack)
6. [Privilege escalation — finding root clues & flags](#privilege-escalation--finding-root-clues--flags)
7. [Root flag & cleanup](#root-flag--cleanup)
8. [References & notes](#references--notes)

---

## 1. Recon and port scan

Start with a targeted `nmap` scan (TCP service detection + common scripts). Example used in this room:

```bash
nmap -sC -sV -p 22,80 -Pn 10.10.4.235
```

Observed services (example output):

* `22/tcp` — SSH (OpenSSH)
* `80/tcp` — HTTP (Apache)

Reference screenshot (nmap output):

![nmap results](/assets/img/TryHackMe/jacktrades/nmap_result.png)

---

## 2. Web discovery and quick probing

Open the web server in the browser and enumerate:

* browse the site root and check typical directories (`/assets`, `/index.php`, etc.)
* view source and try simple directory listing or `cmd` parameter (if present)

Site header screenshot and site accessed:

![site header](/assets/img/TryHackMe/jacktrades/site_header.png)
![site accessed](/assets/img/TryHackMe/jacktrades/site_accesed.png)

While exploring, a page had a GET `cmd` parameter that blindly ran shell commands and returned output — a RCE vector. We'll exploit that next.

---

## 3. Remote command execution (RCE) via cmd parameter

The application had a page like:

```
http://<target>/nnxhweOV/index.php?cmd=<command>
```

We used this to run harmless commands first (`ls -al /home`) and confirmed output was returned in the page. Example screenshot where `cmd=cat /home/jacks_password_list` was used:

![RCE demonstration](/assets/img/TryHackMe/jacktrades/RCE.png)

Once confirmed, dump relevant files using `cmd` (e.g. `cat /home/jacks_password_list`).

---

## 4. Harvesting jacks_password_list and decoding

The `jacks_password_list` file returned lots of junk/encoded-looking lines (screenshot):

![password list](/assets/img/TryHackMe/jacktrades/passowrd_list.png)

From inspection, multiple lines looked like different encodings. Typical steps:

1. Save the output locally (`curl` or copy/paste).
2. Try simple decoders (ROT13, Base64, ROT47, or an online reversible obfuscation tool) on interesting lines.
3. Identify the valid username/password combination embedded in the file.

The extracted credentials (example screenshot showing retrieved credentials from steghide/header or the list):

![passwords found](/assets/img/TryHackMe/jacktrades/passwords.png)
*(This illustration shows the discovered credentials — use them only in the challenge environment.)*

---

## 5. Initial access — get a shell as jack

We used the retrieved credentials to SSH to the host as `jack` (or, in some workflows, the web app allowed spawning a reverse shell — the room used direct SSH with credentials recovered). After logging in as `jack`, check `~/` for flags:

```bash
ssh jack@10.10.x.y
ls -la
cat user.txt
```

Example: the user flag screenshot:

![user flag](/assets/img/TryHackMe/jacktrades/user_flag.png)

---

## 6. Privilege escalation — finding root clues & flags

After initial shell as `jack`, perform standard enumeration:

* `sudo -l` to check allowed sudo commands
* search for SUID binaries, world-writable files, interesting scripts, or readable root files.
* `find / -perm -4000 -type f 2>/dev/null` to list SUID binaries.

A useful clue in this room: a file `/root/root.txt` was readable via `strings` from a binary or via certain helper tools. Another vector was a document (e.g. site assets) that contained credentials or a passphrase for an embedded archive.

Screenshots documenting escalation steps:

* found SUID/interesting binaries and used `strings` (example):
  ![strings/root reading](/assets/img/TryHackMe/jacktrades/root_pass.png)

* root flag captured:
  ![root flag](/assets/img/TryHackMe/jacktrades/root_flag.png)

**Important methods used** (summary):

* `sudo -l` (check allowed commands)
* SUID binary exploitation (when present)
* `strings /usr/bin/somebinary` to reveal embedded secrets
* reading leftover notes/todo files that had hints

---

## 7. Root flag & cleanup

Once you obtain root (via exploiting a SUID helper or allowed `sudo` command), collect root flag:

```bash
sudo -i   # or other privilege escalation method performed
cat /root/root.txt
```

Store the flag, then perform minimal cleanup (only in lab environment): close shells and remove any uploaded files if you uploaded any payloads during exploitation.

Final root flag screenshot:

![root flag](/assets/img/TryHackMe/jacktrades/root_flag.png)

---

## 8. References & notes

* Enumerate services first (nmap)
* Carefully inspect web app for parameters that execute commands
* When you find encoded/obfuscated password lists, try:

  * Base64 / ROT13 / ROT47
  * Online or offline decoders (CyberChef is great)
* For privilege escalation, `sudo -l` and SUID checks are high-value
* Use `strings` to inspect binaries for embedded secrets or notes

---
