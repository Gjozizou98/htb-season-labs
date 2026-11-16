Hack The Box — Penetration Testing Write-ups

Documented walk-throughs of retired Hack The Box machines, written in English from reconnaissance to privilege escalation and remediation. Each write-up focuses on the reasoning behind every step — why a technique works, how a defender would detect it, and how it should be fixed — not just the commands.

All testing was performed in Hack The Box's authorized lab environment. No real or unauthorized systems were targeted.

Author: G. Iannaiello — M.Sc. Cybersecurity student focused on offensive security. Training daily on Hack The Box, TryHackMe and PortSwigger Web Security Academy.

Machines
Machine	OS	Difficulty	Key techniques	Write-up
Shocker	Linux	Easy	Shellshock (CVE-2014-6271), sudo/perl privesc	Shocker.md
Nibbles	Linux	Easy	CMS enum, weak creds, unrestricted file upload → RCE, sudo misconfig	Nibbles.md
Bashed	Linux	Easy	Exposed debug webshell (phpbash), lateral move (sudo -u), root cron privesc	Bashed.md
Jerry	Windows	Easy	Tomcat Manager, default creds, malicious WAR deploy → SYSTEM	Jerry.md
Netmon	Windows	Easy	Anonymous FTP, config credential hunting, PRTG CVE-2018-9276 → SYSTEM	Netmon.md
Blue	Windows	Easy	SMB enumeration, EternalBlue (MS17-010) → SYSTEM	Blue.md
Skills demonstrated
Reconnaissance & enumeration — nmap (full-port, service/version, NSE vuln scripts), web content discovery (gobuster), SMB/FTP enumeration.
Web exploitation — unrestricted file upload, CMS vulnerabilities, authenticated RCE, information disclosure.
Credential attacks — config/credential hunting, weak-password analysis, lockout-aware targeted guessing.
Privilege escalation — sudo misconfigurations, cron jobs, Windows service-account abuse, kernel exploits.
Shells & tooling — reverse shells (bash /dev/tcp, Python), evil-winrm, Metasploit, payload generation with msfvenom.
Reporting — client-style findings with severity ratings, impact and remediation (see the sample report).
Methodology

Every engagement follows a consistent, repeatable flow:

Recon → Enumeration → Exploitation (foothold) → Shell stabilization → Privilege escalation → Loot & report.

The guiding principle throughout: read what the target tells you (source comments, error pages, config files, version banners) and verify before exploiting (e.g. confirm MS17-010 with an NSE check before firing EternalBlue) rather than firing tools blindly.

Related
Sample penetration test report — a client-format report (executive summary, severity-rated findings, remediation): Sample-Pentest-Report.md
Offensive Python tools — small security tools built from scratch: (see the offensive-python-lab repository)
