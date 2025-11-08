---
title: "Jack-of-all-trades — TryHackMe Walkthrough"
date: 2025-11-06
author: shivam
categories: [ctf, tryhackme, writeup]
tags: [jack-of-all-trades, TryHackMe, RCE, privilege-escalation]
summary: "Full walkthrough for the TryHackMe 'jack-of-all-trades' room — recon, web RCE, credential harvesting, shell, and privilege escalation."
image:
  path: /assets/img/TryHackMe/jacktrades/site_header.png
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

## 1. Recon and port scan {#recon-and-port-scan}

Start with an `nmap` scan to discover reachable services and versions. Example command used for this room:

```bash
nmap -sC -sV -p 22,80 -Pn 10.10.4.235
````

Typical results to expect:

* `22/tcp` — OpenSSH (SSH)
* `80/tcp` — Apache HTTP (webapp)

Screenshot (nmap output):

![nmap results](/assets/img/TryHackMe/jacktrades/nmap_result.png)

---

## 2. Web discovery and quick probing {#web-discovery-and-quick-probing}

Browse the web server, inspect `/index.php` and `/assets`, and view page source. Look for parameters that might be dangerous (e.g. `cmd`, `exec`, file includes).

Example site views:

![site header](/assets/img/TryHackMe/jacktrades/site_header.png)
![site accessed](/assets/img/TryHackMe/jacktrades/site_accesed.png)

While poking around the web app we found a `cmd` GET parameter that executed shell commands server-side and returned the output — a classic RCE vector.

---

## 3. Remote command execution (RCE) via cmd parameter {#remote-command-execution-rce-via-cmd-parameter}

The vulnerable endpoint looked like:

```
http://<host>/nnxhweOV/index.php?cmd=<command>
```

First, validate by running safe commands: `ls -al /home`, `whoami`. If output returns in the page, you've got remote command execution.

Screenshot showing `cmd=cat /home/jacks_password_list`:

![RCE demonstration](/assets/img/TryHackMe/jacktrades/RCE.png)

From here, dump files of interest (home directories, config, assets).

---

## 4. Harvesting jacks_password_list and decoding {#harvesting-jacks_password_list-and-decoding}

The file `jacks_password_list` contained many obfuscated/encoded entries. Save the content, and try common decoders against lines that look like encoded data.

Screenshot of the raw list returned:

![password list](/assets/img/TryHackMe/jacktrades/passowrd_list.png)

Common decoding steps to try:

* Base64 decode
* ROT13 / ROT47
* URL decode
* Try CyberChef recipes for mixed encodings

After decoding and testing candidates, you will recover credentials. Example screenshot showing discovered credentials:

![passwords found](/assets/img/TryHackMe/jacktrades/passwords.png)

---

## 5. Initial access — get a shell as jack {#initial-access--get-a-shell-as-jack}

With valid credentials, SSH to the host:

```bash
ssh jack@<target-ip>
# or use a web-based reverse-shell if the app supports it
```

Check the home directory for `user.txt` and other clues:

```bash
ls -la
cat user.txt
```

Example (user flag retrieved):

![user flag](/assets/img/TryHackMe/jacktrades/user_flag.png)

---

## 6. Privilege escalation — finding root clues & flags {#privilege-escalation--finding-root-clues--flags}

Standard enumeration steps as `jack`:

* `sudo -l` (check allowed `sudo` commands)
* `find / -perm -4000 -type f 2>/dev/null` (list SUID binaries)
* Search for readable root-owned files, important service configs or leaked credentials

In this room we found useful data via `strings` on binaries or notes in assets and found root-related clues. Example screenshot of finding a root password / note:

![root password/clue](/assets/img/TryHackMe/jacktrades/root_pass.png)

Once a viable escalation path is identified (SUID exploit, sudo misconfiguration, leaked password), perform the escalation and validate root.

---

## 7. Root flag & cleanup {#root-flag--cleanup}

After obtaining root, read the root flag:

```bash
sudo -i
cat /root/root.txt
```

Root flag screenshot:

![root flag](/assets/img/TryHackMe/jacktrades/root_flag.png)

After finishing (lab environment only), tidy up any files or shells you created.

---

## 8. References & notes {#references--notes}

* Use `nmap` for initial enumeration.
* Use browser dev tools + view-source to find `cmd` or suspicious parameters.
* Decoding workflows: CyberChef is extremely helpful for mixed encodings.
* For privilege escalation: `sudo -l`, SUID checks, `strings` on binaries, and searching for leftover notes are high-value techniques.

---
