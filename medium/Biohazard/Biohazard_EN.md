# Biohazard — TryHackMe
 
> **Difficulty:** Medium  
> **Category:** Web Enumeration · Cryptography · Steganography · Privilege Escalation  
> **Date:** 2025-07-12  
> **Room:** https://tryhackme.com/room/biohazard
 
---
 
## overview
 
Resident Evil-themed room. The goal is not to exploit a service vulnerability — it's to follow a long chain of clues hidden across web directories, decode layered ciphers, extract data via image steganography, and only then gain SSH access. The final privilege escalation to root comes from a critical sudo misconfiguration. The room demands attention to detail from the very start: information collected early becomes a decryption key much later.
 
---
 
## reconnaissance
 
```bash
nmap -sC -sV -p- <TARGET_IP>
```
 
| port | service | version |
|---|---|---|
| 21/tcp | ftp | vsftpd 3.0.3 |
| 22/tcp | ssh | OpenSSH 7.6p1 |
| 80/tcp | http | Apache |
 
FTP with no anonymous login. SSH with no credentials yet. Entry point: port 80.
 
---
 
## enumeration
 
### Port 80 — /mansionmain/
 
The home page has a link to `/mansionmain/`. Inspecting the source code:
 
```html
<!-- It is in the /diningRoom/ -->
```
 
The mansion starts talking. We follow.
 
---
 
### /diningRoom/ — emblem flag + base64 hint
 
The dining room has a link to take the emblem on the wall — this yields the **emblem flag**.
 
The source code also contains a Base64 string:
 
```
SG93IGFib3V0IHRoZSAvdGVhUm9vbS8=
```
 
Decoding it:
 
```bash
echo "SG93IGFib3V0IHRoZSAvdGVhUm9vbS8=" | base64 -d
# How about the /teaRoom/
```
 
---
 
### /teaRoom/ — lockpick flag
 
The tea room contains the **lockpick flag** and a note telling Jill to visit `/artRoom/`.
 
---
 
### /artRoom/ — mansion map
 
The art room reveals a map with all available directories:
 
```
/diningRoom/    /teaRoom/       /artRoom/      /barRoom/
/diningRoom2F/  /tigerStatusRoom/  /galleryRoom/
/studyRoom/     /armorRoom/     /attic/
```
 
Full roadmap. From here: systematic enumeration room by room.
 
---
 
### /barRoom/ — Base32 → music sheet flag → gold emblem → "rebecca"
 
The bar room door is locked. We use the **lockpick flag** to enter.
 
Inside there is a note with text encoded in **Base32**. Decoding via CyberChef:
 
```
Base32 → music sheet flag
```
 
Submitting the music sheet flag at the piano reveals a **gold emblem** on the wall.
 
Trying the **gold emblem** directly in the gold emblem slot does nothing. However, submitting the **original dining room emblem** in the same slot redirects to a page revealing the name **"rebecca"** — noted for later use as a decryption key.
 
---
 
### /diningRoom/ — gold emblem slot → Vigenere → shield key
 
Submitting the **gold emblem** in the dining room returns a ciphertext. It looks like ROT13 but it isn't — it's a **Vigenere Cipher** with the key **"rebecca"** (obtained in the previous step).
 
Decoding:
 
```
Vigenere (key: rebecca) → "there is a shield key inside the dining room. The html page is called the_great_shield_key"
```
 
Navigating to `/diningRoom/the_great_shield_key.html` gives us the **shield key flag**.
 
---
 
### /diningRoom2F/ — ROT13 → blue jewel
 
The page source contains a comment:
 
```
Lbh trg gur oyhr trz ol chfuvat gur fgnghro gb gur ybjre sybbe...
```
 
Applying **ROT13**:
 
```
"You get the blue gem by pushing the status to the lower floor. Visit sapphire.html"
```
 
Navigating to `/diningRoom/sapphire.html` gives us the **blue jewel flag**.
 
---
 
### Collecting the 4 Crests
 
With the collected items, the remaining rooms unlock. Each one delivers a crest with a different encoding. Tool used: **CyberChef**.
 
**Crest 1 — /tigerStatusRoom/**
Blue jewel unlocks the room. Crest encoded twice: **Base64 → Base32**.
 
```
Crest 1: RlRQIHVzZXI6IG
```
 
**Crest 2 — /galleryRoom/**
Examining the note in the gallery. Crest encoded in **Base32 → Base58**.
 
```
Crest 2: h1bnRlciwgRlRQIHBh
```
 
**Crest 3 — /armorRoom/**
Shield key unlocks. Crest encoded three times: **Base64 → Binary → Hex → ASCII**.
 
```
Crest 3: c3M6IHlvdV9jYW50X2g
```
 
**Crest 4 — /attic/**
Shield key unlocks. Crest encoded in **Base58 → Hex**.
 
```
Crest 4: pZGVfZm9yZXZlcg==
```
 
---
 
### Combining the 4 Crests → FTP credentials
 
Concatenating all four crests:
 
```
RlRQIHVzZXI6IGh1bnRlciwgRlRQIHBhc3M6IHlvdV9jYW50X2hpZGVfZm9yZXZlcg==
```
 
Decoding from **Base64**:
 
```bash
echo "RlRQIHVzZXI6IGh1bnRlciwgRlRQIHBhc3M6IHlvdV9jYW50X2hpZGVfZm9yZXZlcg==" | base64 -d
# FTP user: hunter, FTP pass: [censured]
```
 
---
 
## exploitation
 
### FTP — steganography in images
 
```bash
ftp hunter@<TARGET_IP>
mget *
```
 
Files downloaded:
 
```
important.txt
001-key.jpg
002-key.jpg
003-key.jpg
helmet_key.txt.gpg
```
 
`important.txt` — a note from Barry to Jill mentioning a `/hidden_closet/` and that the helmet key is somewhere in the files.
 
The three images carry hidden data via steganography:
 
**001-key.jpg — steghide with no passphrase:**
 
```bash
steghide extract -sf 001-key.jpg
# extracts key-001.txt → cGxhbnQ0Ml9jYW
```
 
**002-key.jpg — strings:**
 
```bash
strings 002-key.jpg | grep -i comment
# Comment: 5fYmVfZGVzdHJveV8
```
 
**003-key.jpg — binwalk (steghide required a passphrase):**
 
```bash
binwalk -e 003-key.jpg
# extracts key-003.txt → 3aXRoX3Zqb2x0
```
 
Concatenating the three parts and decoding from **Base64**:
 
```bash
echo "cGxhbnQ0Ml9jYW5fYmVfZGVzdHJveV93aXRoX3Zqb2x0" | base64 -d
# result: [censured]
```
 
This is the passphrase to decrypt the GPG file:
 
```bash
gpg helmet_key.txt.gpg
# passphrase: [censured]
# result: helmet key flag
```
 
---
 
### /studyRoom/ + /hidden_closet/ → SSH credentials
 
With the **helmet key flag**, we unlock the Study Room.
 
Inside there is a `doom.tar.gz` file available for download:
 
```bash
gunzip doom.tar.gz
tar -xf doom.tar
# extracts eagle_medal.txt → SSH user: umbrella_[censured]
```
 
In `/hidden_closet/`, using the helmet key, we find **MO Disk 1** with a message encoded in **Vigenere** (albert):
 
```
wpbwbxr wpkzg pltwnhro, txrks_xfqsxrd_bvv_fy_rvmexa_ajk
→ albert weasker password: [censured]
```
 
Still in the hidden closet, the **wolf medal** reveals:
 
```
SSH password: [censured]
```
 
SSH credentials obtained:
 
```
username: umbrella_[censured]
password: [censured]
```
 
---
 
### SSH — initial access and pivot to weasker
 
```bash
ssh umbrella_guest@<TARGET_IP>
```
 
Inside the home directory there is a hidden `.jailcell` folder containing `chris.txt` — story content. At the end of the file:
 
```
MO disk 2: albert
```
 
Checking other users in `/home`: **hunter** and **weasker**.
 
Switching to weasker with the password obtained from MO Disk 1:
 
```bash
su weasker
# password: [censured]
```
 
---
 
## privilege escalation
 
### Post-access enumeration
 
```bash
id
```
 
```
uid=1000(weasker) gid=1000(weasker) groups=1000(weasker),4(adm),24(cdrom),27(sudo)...
```
 
`sudo` group present — immediate check:
 
```bash
sudo -l
```
 
```
User weasker may run the following commands on umbrella_corp:
    (ALL : ALL) ALL
```
 
### Vulnerability analysis
 
`(ALL : ALL) ALL` means: any command, as any user, without restriction. Weasker has full administrative control. No additional exploit needed — the misconfiguration is the path.
 
> **Vulnerability:** Sudo Privilege Misconfiguration  
> **MITRE ATT&CK:** T1548.003 — Abuse Elevation Control Mechanism: Sudo  
> **Severity:** Critical
 
### Execution
 
```bash
sudo su
```
 
```
root@umbrella_corp:~#
```
 
---
 
## flags
 
```bash
cat /root/root.txt
# flag: [censured]
```
 
---
 
## attack chain
 
```
Nmap → 21 (FTP) · 22 (SSH) · 80 (HTTP)
          │
          ▼
/mansionmain/ → source code → /diningRoom/
          │
          ▼
emblem flag + Base64 → /teaRoom/
          │
          ▼
lockpick flag → /artRoom/ → mansion map
          │
          ▼
/barRoom/ (lockpick) → Base32 → music sheet → gold emblem
          │
          ▼
original emblem in bar room slot → name "rebecca"
          │
          ▼
gold emblem in diningRoom → Vigenere (key: rebecca) → shield key
          │
          ▼
/diningRoom2F/ → ROT13 → blue jewel
          │
          ▼
4 Crests (Base64·Base32 / Base32·Base58 / Base64·Bin·Hex / Base58·Hex)
          │
          ▼
Crests concatenated → Base64 → FTP: hunter / you_cant_hide_forever
          │
          ▼
FTP → 3 images → steghide + strings + binwalk → GPG key
          │
          ▼
GPG decrypt → helmet key flag
          │
          ▼
/studyRoom/ → doom.tar.gz → SSH user: umbrella_guest
/hidden_closet/ → Vigenere autosolve → weasker password
                → wolf medal → SSH pass: T_virus_rules
          │
          ▼
SSH (umbrella_guest) → .jailcell/chris.txt → MO disk 2: albert
          │
          ▼
su weasker (stars_members_are_my_guinea_pig)
          │
          ▼
sudo -l → (ALL : ALL) ALL → sudo su → root
          │
          ▼
root.txt → flag captured
```
 
---
 
## what I learned
 
This room has a chain too long to solve without notes. The name **"rebecca"** only shows up mid-way through the bar room section — anyone who missed it got stuck at the Vigenere step. This mirrors real engagement behavior: information that seems decorative early on often becomes critical later.
 
Another relevant point: the three extraction techniques used on the images (steghide, strings, binwalk) covered different scenarios. Steghide required a passphrase on one of the images, which forced the switch to binwalk as an alternative. In real pentesting, having more than one tool for the same objective is essential.
 
The sudo escalation was trivial after all the complexity of the chain. This happens: environments with unrestricted sudo for a compromised user hand over root immediately, regardless of how complex the initial access path was.
 
**Remediation:**
 
```bash
# Insecure configuration found
weasker ALL=(ALL:ALL) ALL
 
# Correct — principle of least privilege
weasker ALL=(root) NOPASSWD: /usr/bin/systemctl restart apache2
```
 
Review `/etc/sudoers` regularly, remove unnecessary users from the `sudo` group, and never leave temporary permissions without an expiration date.
