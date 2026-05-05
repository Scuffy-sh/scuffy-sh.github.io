---
layout: single
title: "HTB NanoCorp - Writeup"
date: 2026-05-05
difficulty: Hard
operating_system: Windows
service_hint: Active Directory + NTLM Relay + Checkmk + MSI Repair
tags:
  - Active Directory
  - NTLM Relay
  - CVE
  - BloodHound
  - Privilege Escalation
  - Checkmk
  - Responder
summary: "Cadena de explotación: NTLM relay vía CVE-2025-24054 para comprometer web_svc, manipulación de grupos AD para acceder a monitoring_svc mediante CVE-2024-0670 en Checkmk, y escalada a SYSTEM usando MSI repair trigger."
---

# Hack The Box: NanoCorp (Hard)

## Summary
NanoCorp is a Windows Server 2022 Active Directory domain controller (nanocorp.htb) rated Hard. The attack path starts with NTLM relay via CVE-2025-24054 to compromise the `web_svc` user, followed by Active Directory group manipulation to gain access to `monitoring_svc` via Checkmk CVE-2024-0670 exploitation, and finally privilege escalation to SYSTEM using an MSI repair trigger.

## Reconnaissance
Initial Nmap scan identified the following open services on 10.129.56.49:
- Port 53: Simple DNS Plus (domain: nanocorp.htb)
- Port 80: Apache 2.4.58 (Win64) with OpenSSL/3.1.3 PHP/8.2.12, redirects to `nanocorp.htb`
- Port 88: Kerberos (Windows AD)
- Ports 139/445: SMB (signing enabled and required)
- Ports 389/3268: LDAP (Active Directory, domain: nanocorp.htb)
- Port 3389: RDP (host: DC01.nanocorp.htb, OS: Windows Server 2022 Build 20348)
- Port 5986: WinRM (SSL, host: dc01.nanocorp.htb)
- Port 6556: Checkmk Agent 2.1.0p10
- Port 9389: ADWS

![Web Redirect](/images/writeups/nanocorp/nanocorp-web-redirect.png)

## Enumeration
- Subdomain `hire.nanocorp.htb` discovered via redirect from the main site. Gobuster enumeration revealed directories like `images`, `assets`, and restricted paths (`uploads`, `phpmyadmin`).
- Kerbrute user enumeration found only `administrator` as an initial valid user.
- Checkmk agent (port 6556) traffic revealed active sessions for the `NANOCORP\web_svc` user.
- After compromising `web_svc`, NetExec LDAP enumeration listed 5 domain users: Administrator, Guest, krbtgt, web_svc, monitoring_svc.
- BloodHound analysis showed `web_svc` could be added to the `IT_SUPPORT` group, which had privileges to modify `monitoring_svc` credentials.

![BloodHound](/images/writeups/nanocorp/nanocorp-bloodhound.png)

## Exploitation
### Initial Access (CVE-2025-24054: NTLM Relay)
1. Created a malicious `.library-ms` file pointing to an attacker-hosted SMB share, zipped it to trigger NTLM authentication from the victim.
2. Started Responder to capture NTLMv2 hashes for `NANOCORP\web_svc`.
3. Cracked the hash using Hashcat with the `rockyou.txt` wordlist, obtaining the password: `dksehdgh712!@#`.
4. Validated credentials via NetExec SMB, confirming `web_svc` access.

![Responder Capture](/images/writeups/nanocorp/nanocorp-responder.png)

## Privilege Escalation
### Step 1: Gain IT_SUPPORT Membership
Used `bloodyAD` to add `web_svc` to the `IT_SUPPORT` group, leveraging BloodHound-identified privileges.

### Step 2: Compromise monitoring_svc
As `IT_SUPPORT` member, reset `monitoring_svc` password to `NewPass123!` via `bloodyAD`. Generated a Kerberos TGT for `monitoring_svc` and used Impacket's `winrmexec.py` to gain a WinRM shell as `monitoring_svc` on port 5986.

### Step 3: User Flag
Retrieved user.txt from `C:\Users\monitoring_svc\Desktop`:
`07009788b4ede13461f0f5d8247322b7`

### Step 4: Escalate to SYSTEM (CVE-2024-0670: Checkmk Agent)
1. Identified Checkmk Agent 2.1.0.50010 (vulnerable to CVE-2024-0670) installed on the host.
2. Compiled `RunasCs` to execute commands as `web_svc` (used to trigger the MSI repair as a privileged user).
3. Deployed `privesc.ps1` which:
   - Sprays read-only `.cmd` payloads (with reverse shell to attacker) in `C:\Windows\Temp`
   - Triggers MSI repair for the Checkmk Agent, which executes the malicious `.cmd` file as SYSTEM
4. Captured reverse shell as `nt authority\system` via netcat listener.

![Privilege Escalation](/images/writeups/nanocorp/nanocorp-privesc.png)

## Flags
- **User Flag**: `07009788b4ede13461f0f5d8247322b7`
- **Root Flag**: `df398fa317da91fe096884185fc2edab`

## Conclusion
Key takeaways from this engagement:
1. NTLM relay attacks via file-based triggers (e.g., `.library-ms`) remain a viable initial access vector for unpatched Windows environments.
2. Over-privileged AD groups (like `IT_SUPPORT`) can enable lateral movement and credential manipulation if not properly scoped.
3. Unpatched third-party software (Checkmk Agent) can provide straightforward privilege escalation paths to SYSTEM.
4. Proper SMB signing, NTLM relay mitigations, and timely patching of CVEs (CVE-2025-24054, CVE-2024-0670) are critical for defense.
