# Penetration Test Report - Active

**Target:** 10.129.102.43 (active.htb)
**Classification:** Internal Active Directory assessment
**Tester:** Giovanni
**Methodology:** Manual enumeration and exploitation, no automated vulnerability scanners

---

## Executive Summary

This assessment targeted a single Windows server acting as a Domain Controller for the active.htb domain. During testing, an anonymously accessible network share was found to contain a file with a recoverable password, which in turn allowed the recovery of the Domain Administrator password through a well-known Active Directory attack technique.

As a result, an attacker with no prior credentials and no privileged network position could gain full administrative control over the entire domain, including every computer and user account within it.

It is recommended that the exposed file be removed, the affected account password be rotated immediately, and SMB share permissions across the domain be reviewed to prevent anonymous access to internal configuration data.

---

## Scope & Methodology

The assessment was limited to the single host at 10.129.102.43. Testing followed a standard black-box approach: no credentials or internal documentation were provided at the start.

The methodology followed these phases:
1. Network reconnaissance to identify exposed services
2. Service enumeration to determine configuration and access levels
3. Exploitation of identified weaknesses to obtain credentials
4. Privilege escalation to the highest level of access available
5. Verification of impact

All testing was performed manually, without automated exploitation frameworks, to ensure every step could be fully understood and explained.

---

## Findings

### Finding 1: Anonymous SMB access to a share containing cleartext-recoverable credentials

**Severity:** High

**Description**
The target exposed multiple SMB shares. While most default administrative shares correctly rejected anonymous access, one share ("Replication") allowed anonymous read access. This share is used by domain controllers to replicate Group Policy data and is expected to be broadly readable within a domain, but its contents should never include recoverable credentials.

Inside this share, a Group Policy Preferences file (`Groups.xml`) was found containing a `cpassword` attribute. This field is used by an older Windows feature that allowed administrators to set local account passwords through Group Policy. The password is encrypted, but with a static AES key that Microsoft published publicly in its documentation (CVE-2014-1812 / MS14-025). As a result, any value found in this field can be decrypted by anyone, instantly.

**Evidence**
- Anonymous access confirmed via SMB enumeration
- File located at: `active.htb\Policies\{GUID}\MACHINE\Preferences\Groups\Groups.xml`
- Associated account: `active.htb\SVC_TGS`
- Password value: `[REDACTED]`

**Business Impact**
This finding allowed an external party with no credentials and no privileged access to obtain a valid domain account. In a production environment, this is equivalent to leaving a door key under the doormat: no technical skill is required to find it, only knowledge of where to look.

**Remediation**
- Remove the legacy GPP password configuration; Microsoft has deprecated this feature since 2014
- Rotate the password of the affected account (`SVC_TGS`) immediately
- Audit all SMB shares for anonymous access and restrict permissions to authenticated, authorized users only

---

### Finding 2: Kerberoasting of the Domain Administrator account

**Severity:** Critical

**Description**
Using the credentials obtained in Finding 1, the domain's Kerberos service was queried for accounts with a registered Service Principal Name (SPN). This revealed that the **Administrator** account itself had an SPN registered for the CIFS service.

Any authenticated domain account, regardless of privilege level, can request a Kerberos service ticket for any account with a registered SPN. This ticket is encrypted using the service account's own password hash. Because the Administrator account had an SPN, its service ticket could be requested and taken offline for password cracking, without generating repeated failed authentication attempts against the domain controller.

The ticket was cracked offline against a common password list, recovering the Administrator account's plaintext password in under one minute.

**Evidence**
- SPN identified: `active/CIFS:445`, associated with the `Administrator` account
- Ticket requested via Kerberoasting and cracked offline
- Resulting password: `[REDACTED]`
- Access confirmed via remote command execution as `active\administrator`

**Business Impact**
This is the most severe finding of the assessment: it represents full compromise of the Active Directory domain. An attacker with this level of access can read, modify, or delete any data in the domain, create new accounts, disable security controls, and move laterally to any connected system. In practice, this is equivalent to handing over the keys to the entire organization's IT infrastructure, not just one server.

**Remediation**
- Remove the SPN from the Administrator account; privileged accounts should never be configured as service accounts
- Enforce a strong, long, unique password policy for all privileged accounts, ideally with a password manager or vault solution
- Monitor for Kerberos ticket requests (TGS-REQ) targeting privileged accounts, as this is a well-known attack pattern
- Consider enabling Managed Service Accounts (MSA/gMSA) for any legitimate service accounts, which rotate passwords automatically and are not kerberoastable in the same way

---

## Conclusion

Both findings stem from the same underlying theme: credential exposure and misconfiguration, rather than a software vulnerability. No exploit was used against unpatched software; every step relied on legitimate Active Directory functionality being misused due to weak configuration choices.

This highlights an important principle for defenders: Active Directory security depends heavily on configuration hygiene, not just patching. A single forgotten legacy setting (GPP passwords) combined with a single privileged account misconfiguration (an SPN on Administrator) was sufficient to compromise the entire domain.

**Overall risk rating: Critical** — full domain compromise was achieved with no prior access and no advanced tooling.
