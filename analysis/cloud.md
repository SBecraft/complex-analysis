---
source: Azure AD sign-in + audit logs (logs/cloud/)
analyst: cloud-analyst
date_analyzed: 2026-07-24
files_reviewed:
  - logs/cloud/signins-2026-07-09.jsonl
  - logs/cloud/signins-2026-07-10.jsonl
  - logs/cloud/signins-2026-07-11.jsonl
  - logs/cloud/audit-2026-07-09.jsonl
  - logs/cloud/audit-2026-07-10.jsonl
  - logs/cloud/audit-2026-07-11.jsonl
---

# Cloud Log Analyst — Incident Summary

Analysis complete across all 6 files (257 sign-in events, 62 audit events, 20 users, 3 days).

## Timeline (all times UTC)

| Time | Event | Source |
|---|---|---|
| 2026-07-09 08:37:20 | `mrodriguez` adds member to group, from 203.0.113.10 (normal corp egress) | audit-07-09 |
| 2026-07-10 07:56:18 | `mrodriguez` signs into SharePoint Online, WKSTN-017, 203.0.113.10 (baseline) | signins-07-10 |
| 2026-07-10 08:12:03 | `mrodriguez` signs into Exchange Online, WKSTN-017, 203.0.113.12 (baseline) | signins-07-10 |
| **2026-07-10 11:00:14** | **`mrodriguez` signs into Exchange Online from 185.220.101.47 (Bucharest, RO) — no device info, `riskLevel=high`, `riskState=atRisk`, `riskEventTypes_v2=[unfamiliarFeatures, anonymizedIPAddress]`, `conditionalAccessStatus=success`** | signins-07-10 |
| 2026-07-10 11:26:37 | `mrodriguez` signs into Exchange Online again, WKSTN-017, 203.0.113.12 (Austin) — 26 min after the Romania sign-in | signins-07-10 |
| **2026-07-10 11:35:22** | **`mrodriguez` registers new security info ("User registered security info") — adds "Microsoft Authenticator" as StrongAuthenticationMethod, initiated from 185.220.101.47** | audit-07-10 |
| **2026-07-10 11:50:47** | **`mrodriguez` grants OAuth consent to service principal "Data Sync Helper", scopes `Mail.Read Mail.ReadWrite Directory.Read.All offline_access`, initiated from 185.220.101.47** | audit-07-10 |
| 2026-07-10 11:44:11 -> 19:01:06 | `mrodriguez` continues normal-looking activity from WKSTN-017 / 203.0.113.x throughout the rest of 07-10 and into 07-11 | signins-07-10/11 |

Everything else in all 6 files (19 other users, ~250 remaining sign-ins, remaining audit events — password self-service resets, group membership adds, "Update application" no-op edits) is routine and shows no other risk flags, no other foreign/anonymized IPs, no other multi-OS/multi-device users, and no other MFA-registration or consent-grant events tied to an anomalous IP.

## IOCs

- **Compromised identity**: `mrodriguez@corp.com`
- **Malicious IP**: `185.220.101.47` — this is a known Tor exit-node range (185.220.101.0/24); appears exactly 3 times in the entire dataset, all for this one user, all within a 51-minute window
- **Legitimate baseline IPs for mrodriguez**: 203.0.113.10, .11, .12 (Austin, TX)
- **Legitimate device**: WKSTN-017 (Windows 10, AzureAD-joined, compliant/managed)
- **Malicious app/service principal**: "Data Sync Helper" — newly consented app, requested `Mail.Read`, `Mail.ReadWrite`, `Directory.Read.All`, `offline_access`
- **Newly registered MFA method**: Microsoft Authenticator, added by the attacker session (not the user's existing method)
- **Tenant**: single-tenant dataset scoped to `corp.com`; no tenantId field present in the log schema
- Note: `userId`/`deviceDetail.deviceId` values in this dataset are regenerated per event rather than stable per-user identifiers — do not use them as correlation keys; use `userPrincipalName`/`deviceDetail.displayName` instead.

## ATT&CK Techniques

| Technique | ID | Evidence |
|---|---|---|
| Valid Accounts: Cloud Accounts | T1078.004 | Sign-in succeeds with no CA block/MFA challenge despite `high` risk and foreign/anonymized IP — consistent with a stolen session/token rather than a fresh credential-based logon |
| Use Alternate Authentication Material (session/token replay) | T1550.001 / T1539 | `riskEventTypes_v2: [unfamiliarFeatures, anonymizedIPAddress]` + `conditionalAccessStatus: success` + impossible travel is the textbook Identity-Protection signature for a replayed session cookie/token (e.g., AiTM phishing) rather than a password guess |
| Proxy: Multi-hop Proxy (Tor) | T1090.003 | Source IP is a Tor exit node |
| Account Manipulation: Additional Cloud Credentials/Device Registration | T1098.005 | Attacker registers a new "Microsoft Authenticator" security-info method on the compromised account 35 minutes after initial access — persistence that survives a token expiry or password reset |
| Use Alternate Authentication Material: Application Access Token (Illicit Consent Grant) | T1550.001 | OAuth consent to unfamiliar app "Data Sync Helper" with `Mail.Read/ReadWrite/Directory.Read.All/offline_access` — `offline_access` grants a durable refresh token independent of the user's own session |
| Collection: Email Collection (staged, not yet observed executing) | T1114.002 | Scopes granted enable mailbox read access; no subsequent Graph/mail activity from the app appears in the 3-day window captured — exfil may be pending or occur after this dataset's window |
| Impossible travel / Initial Access indicator | (supporting) | Austin (203.0.113.12, 08:12 UTC) -> Bucharest (185.220.101.47, 11:00 UTC) -> Austin (203.0.113.12, 11:26 UTC): a ~9,600 km round trip in under 3 hours, physically impossible |

## Confidence

- **High** — Account compromise of `mrodriguez@corp.com` via session/token replay on 2026-07-10 ~11:00-11:51 UTC. Basis: single anomalous IP in an otherwise 4-IP/20-user dataset, high risk score with two independent risk-detection signals, zero device metadata (unregistered/attacker infrastructure), impossible travel bracketing it on both sides, and a coherent 3-step attacker sequence (session access -> MFA persistence -> OAuth persistence) all from the identical IP within 51 minutes.
- **Medium** — Whether the underlying compromise is AiTM/phishing-based session-cookie theft vs. a leaked primary refresh token; cloud logs alone can show the token was used anomalously but not how it was obtained.
- **Medium** — Whether "Data Sync Helper" has actually been used to exfiltrate mail; no Graph API/mail-access events for that app ID appear in the 3-day window, so the consent grant is confirmed but downstream abuse is not yet evidenced in this dataset.
- **Low/none** — No evidence found implicating any of the other 19 users, no role/directory-role assignment events at all in this dataset (only group membership, password resets, app registration no-ops), and no other IP outside the 4 seen ever appears.

## Correlation Hints for Endpoint Logs

- Pull Windows Security/Sysmon for host **WKSTN-017** (mrodriguez's registered device) for **2026-07-10 07:00-12:30 UTC**:
  - Look for phishing-delivered content or an AiTM redirect around/before 08:12-11:00 UTC (Outlook message opened, browser navigation to a non-Microsoft login page, unusual child processes off `outlook.exe`/`chrome.exe`/`msedge.exe`).
  - Check for browser credential/cookie access (e.g., processes reading `%LOCALAPPDATA%\...\Cookies` or token cache files) which would support session-token theft as the access vector.
  - Confirm WKSTN-017 itself never made outbound contact to `185.220.101.47` — if it didn't, that supports "session used from attacker infrastructure" rather than "device compromised directly," narrowing this to a cloud-only session hijack.
- Search endpoint/proxy/DNS logs tenant-wide for any contact with `185.220.101.47` or other Tor-exit ranges around 2026-07-10 10:30-12:00 UTC.
- Search mail-flow/EDR for any inbound phishing email to `mrodriguez@corp.com` in the hours preceding 11:00 UTC on 2026-07-10 (subject/sender consistent with a credential-harvesting or consent-phishing lure, given the immediate OAuth consent grant that followed).
- Cross-reference username `mrodriguez` in endpoint logon events (Security 4624/4625, Sysmon) for anomalous interactive logons, new process trees, or lateral movement originating around 11:00-12:00 UTC on 2026-07-10.
- If EDR/mail telemetry supports it, check whether the "Data Sync Helper" app subsequently made any Graph API calls (mailbox reads) after 11:50:47 UTC — this dataset's window ends 2026-07-11 and shows none, but a wider window may.
