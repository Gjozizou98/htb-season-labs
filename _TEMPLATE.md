<!--
  REUSABLE HTB WRITEUP TEMPLATE
  Copy this file, rename it <MachineName>.md, and fill each section.
  Rules of thumb:
    - Explain the WHY, not just the commands (that's what proves understanding).
    - Always include a Remediation section (consultant mindset, not script-kiddie).
    - REDACT flags ([REDACTED]) and use <TARGET_IP> / <MY_IP> placeholders.
    - Only publish writeups for RETIRED machines.
-->

# <MachineName> — Hack The Box

| | |
|---|---|
| **Platform** | Hack The Box |
| **Difficulty** | <Easy / Medium / Hard> |
| **OS** | <Linux / Windows> |
| **Key vulnerability** | <name + CVE if any> |
| **Techniques** | <tag · tag · tag> |

---

## Summary

<2–4 sentences: what the box is, the core vuln, and the attack chain in one line.>

**Attack chain:** `recon` → `enumeration` → `exploit` → foothold (user) → privesc (root).

---

## 1. Reconnaissance

```bash
sudo nmap -p- --min-rate 1000 -sV <TARGET_IP>
```

<Paste the open ports. Note anything unusual (non-standard ports, versions).>

---

## 2. Enumeration

<Per-service enumeration. What you looked for and what you found. Commands + key output.>

---

## 3. Exploitation

**Root cause.** <Explain the vulnerability mechanism — what's actually broken and why.>

**Proof of concept.**
```bash
<the minimal command that confirms the vuln>
```
<Show the evidence that proves it. Break down the payload piece by piece.>

---

## 4. Foothold — user flag

<How you turned the exploit into interactive access (reverse shell, creds, etc.).>

```bash
cat /home/<user>/user.txt
[REDACTED]
```

---

## 5. Privilege Escalation — root flag

<Enumeration that found the privesc vector (sudo -l, SUID, cron, group, kernel...).>
<The exploitation + why it works.>

```bash
id            # uid=0(root)
cat /root/root.txt
[REDACTED]
```

---

## 6. Remediation

- <How to fix the entry vulnerability.>
- <How to fix the privesc.>
- <Hardening / least-privilege recommendations.>

---

## Key Takeaways

- <Lesson 1 — a reusable principle, not a box-specific fact.>
- <Lesson 2>
- <Lesson 3>

---

*Writeup by Giovanni ([@Gjozizou98](https://github.com/Gjozizou98)) — retired HTB machine, for educational purposes.*
