# HTB Season Labs — Writeups & Attack-Chain Analysis

Methodological write-ups for **retired** Hack The Box machines, focused on the *why* behind each step: vulnerability mechanisms, exploitation reasoning, and remediation — not just command dumps.

> ⚠️ All writeups cover **retired** machines only. Flags are redacted.

## Approach

Every box follows the same methodology and is documented end-to-end:

`Reconnaissance → Enumeration → Exploitation → Foothold → Privilege Escalation → Remediation`

Each writeup explains the underlying mechanism of every technique and closes with **remediation** and reusable **takeaways** — the mindset of a consultant, not just an attacker.

## Machines

| Machine | OS | Difficulty | Key vulnerability / techniques |
|---|---|---|---|
| [Shocker](./Shocker.md) | Linux | Easy | Shellshock (CVE-2014-6271) · CGI RCE · sudo/GTFOBins privesc |

*(more coming — one machine added per session)*

## Skills demonstrated

- Network & service enumeration (`nmap`, `gobuster`)
- Web & CGI exploitation, CVE analysis
- Reverse shells, Linux privilege escalation (`sudo -l`, GTFOBins, SUID, groups)
- Clear technical writing and remediation guidance

## Tooling

`Kali Linux` · `nmap` · `gobuster` · `curl` · `netcat` · `Burp Suite` · GTFOBins

---

*Maintained by Giovanni ([@Gjozizou98](https://github.com/Gjozizou98)) — aspiring offensive security engineer.*

A reusable [`_TEMPLATE.md`](./_TEMPLATE.md) is included for consistent writeups.
