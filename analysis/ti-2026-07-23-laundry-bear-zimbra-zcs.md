---
source_url: https://www.cisa.gov/news-events/cybersecurity-advisories/aa26-204a
advisory_id: AA26-204A
extraction_date: 2026-07-23
threat_actor: "LAUNDRY BEAR (aka Void Blizzard, CL-STA-1114, TA488)"
campaign: "Ulej / Flowerbed — Zimbra Collaboration Suite (ZCS) webmail exfiltration"
---

# Threat Intelligence Report: LAUNDRY BEAR Zimbra (ZCS) Campaign

## Threat Overview

**Actor:** LAUNDRY BEAR — a Russian state-supported APT, first publicly named by
the Dutch AIVD/MIVD in May 2025. Also tracked as **Void Blizzard** (Microsoft),
**CL-STA-1114** (Palo Alto Unit 42), and **TA488** (Proofpoint, formerly
UNK_PitStop).

**Targets:** Western government and commercial organizations, including the
Defense Industrial Base, federal/local government, education, energy, law
enforcement, media, NGOs, and technology sectors. Reporting notes a pattern of
targeting Ukrainian organizations first, apparently as a testbed before
broader NATO-country deployment.

**Time period:** Group activity as early as April 2024; the ZCS-specific
campaign described in this advisory began **around July 2025** and was
ongoing as of publication (July 2026). The exploited vulnerability was a
zero-day at first use, patched by Synacor in November 2025 as
[CVE-2025-66376](https://www.cve.org/CVERecord?id=CVE-2025-66376).

**Intent:** Espionage — covert, persistent acquisition of email content,
credentials, and directory data, attributed to the Russian Federation. No
financial extortion or destructive impact observed.

**Summary of mechanism:** A view-based (zero-click-adjacent — requires only
viewing the email, not clicking anything) exploit of a stored XSS
vulnerability in ZCS webmail (CVE-2025-66376, CWE-79) delivered via malicious
HTML email. On view, a JavaScript payload smuggled inside an SVG `onload`
attribute runs in the victim's authenticated webmail session, using the
Zimbra SOAP API to harvest the last 90 days of email, the Global Address
List, saved autocomplete passwords, 2FA scratch codes, and to establish
persistence (enabling IMAP, minting a new "ZimbraWeb" Application Passcode).
Data is exfiltrated over DNS and HTTPS to actor-controlled VPS infrastructure
running a custom Python/Docker stack ("Flowerbed" / "Catcher").

---

## TTPs (MITRE ATT&CK, Enterprise v19)

### Reconnaissance
| ID | Technique | Use in campaign | Confidence |
|---|---|---|---|
| [T1595](https://attack.mitre.org/versions/v19/techniques/T1595/) | Active Scanning | Likely used to port-scan for public-facing ZCS instances | Low (assessed as "likely" by CISA, not directly observed) |
| [T1596.005](https://attack.mitre.org/versions/v19/techniques/T1596/005/) | Search Open Technical Databases: Scan Databases | Fingerprinting datasets from commercial vendors to find exploitable ZCS targets | Low |
| [T1593](https://attack.mitre.org/versions/v19/techniques/T1593/) | Search Open Websites/Domains | OSINT to compile target email addresses | Low |
| [T1597](https://attack.mitre.org/versions/v19/techniques/T1597/) | Search Closed Sources | Reuse of previously exfiltrated data to build target lists | Low |
| [T1597.002](https://attack.mitre.org/versions/v19/techniques/T1597/002/) | Search Closed Sources: Purchase Technical Data | Commercial data broker datasets used for target development | Low |
| [T1589.002](https://attack.mitre.org/versions/v19/techniques/T1589/002/) | Gather Victim Identity Info: Email Addresses | Compiling victim email addresses to target with the exploit | Medium |

### Resource Development
| ID | Technique | Use in campaign | Confidence |
|---|---|---|---|
| [T1583](https://attack.mitre.org/versions/v19/techniques/T1583/) | Acquire Infrastructure | Mullvad VPN used to mask identity when managing infrastructure | High |
| [T1583.003](https://attack.mitre.org/versions/v19/techniques/T1583/003/) | Acquire Infrastructure: VPS | VPS procured from multiple providers, often with fabricated KYC identities | High |
| [T1587.001](https://attack.mitre.org/versions/v19/techniques/T1587/001/) | Develop Capabilities: Malware | Custom "Ulej" collection/exfil capability | High |
| [T1587.004](https://attack.mitre.org/versions/v19/techniques/T1587/004/) | Develop Capabilities: Exploits | Novel XSS exploit for CVE-2025-66376, zero-day at first use | High |
| [T1588.002](https://attack.mitre.org/versions/v19/techniques/T1588/002/) | Obtain Capabilities: Tool | Modified Evilginx2 used in prior (pre-ZCS) AiTM campaigns | High (prior campaigns) |
| [T1588.007](https://attack.mitre.org/versions/v19/techniques/T1588/007/) | Obtain Capabilities: AI | Flowerbed codebase shows indications of AI-assisted development | Medium |
| [T1608](https://attack.mitre.org/versions/v19/techniques/T1608/) | Stage Capabilities | Flowerbed (Catcher/Certbot/Nginx/Gardener Docker containers) deployed to VPS | High |

### Initial Access
| ID | Technique | Use in campaign | Confidence |
|---|---|---|---|
| [T1566](https://attack.mitre.org/versions/v19/techniques/T1566/) | Phishing | Malicious email sent to targets containing the exploit | High |
| [T1199](https://attack.mitre.org/versions/v19/techniques/T1199/) | Trusted Relationship | Since ~Nov 2025, payloads sent from compromised victim mail accounts | High |
| [T1078](https://attack.mitre.org/versions/v19/techniques/T1078/) | Valid Accounts | Stolen/purchased credentials used for account access (broader group TTP) | High |

### Execution
| ID | Technique | Use in campaign | Confidence |
|---|---|---|---|
| [T1203](https://attack.mitre.org/versions/v19/techniques/T1203/) | Exploitation for Client Execution | XSS in ZCS executes attacker JS merely by viewing the email | High |

### Persistence
| ID | Technique | Use in campaign | Confidence |
|---|---|---|---|
| [T1098](https://attack.mitre.org/versions/v19/techniques/T1098/) | Account Manipulation | Enables IMAP access, mints "ZimbraWeb" Application Passcode | High |
| [T1556.006](https://attack.mitre.org/versions/v19/techniques/T1556/006/) | Modify Authentication Process: MFA | Creates app-specific passcode + harvests 2FA scratch codes to bypass MFA | High |

### Privilege Escalation
| ID | Technique | Use in campaign | Confidence |
|---|---|---|---|
| [T1078](https://attack.mitre.org/versions/v19/techniques/T1078/) | Valid Accounts | Reuse of harvested credentials for continued/escalated access | High |

### Defense Evasion
| ID | Technique | Use in campaign | Confidence |
|---|---|---|---|
| [T1027.010](https://attack.mitre.org/versions/v19/techniques/T1027/010/) | Obfuscated Files or Information: Command Obfuscation | Non-functional @import directives added to defeat signatures | High |
| [T1027.013](https://attack.mitre.org/versions/v19/techniques/T1027/013/) | Obfuscated Files or Information: Encrypted/Encoded File | Base64 outer payload + XOR-encrypted inner payload | High |
| [T1027.017](https://attack.mitre.org/versions/v19/techniques/T1027/017/) | Obfuscated Files or Information: SVG Smuggling | Payload hidden in an SVG `onload` attribute | High |
| [T1550.004](https://attack.mitre.org/versions/v19/techniques/T1550/004/) | Use Alternate Authentication Material: Web Session Cookie | Prior AiTM campaigns replayed stolen session cookies | Medium (prior campaigns) |

### Credential Access
| ID | Technique | Use in campaign | Confidence |
|---|---|---|---|
| [T1589.001](https://attack.mitre.org/versions/v19/techniques/T1589/001/) | Gather Victim Identity Info: Credentials | Injects hidden login fields to harvest password-manager autocomplete values | High |
| [T1556.006](https://attack.mitre.org/versions/v19/techniques/T1556/006/) | Modify Authentication Process: MFA | (see Persistence) — also credential-access relevant | High |
| [T1557](https://attack.mitre.org/versions/v19/techniques/T1557/) | Adversary-in-the-Middle | Evilginx2-based AiTM in prior (pre-ZCS) campaigns | Medium (prior campaigns) |

### Discovery
| ID | Technique | Use in campaign | Confidence |
|---|---|---|---|
| [T1087](https://attack.mitre.org/versions/v19/techniques/T1087/) | Account Discovery | Brute-forces the Global Address List via SOAP `SearchGalRequest` | High |
| [T1185](https://attack.mitre.org/versions/v19/techniques/T1185/) | Browser Session Hijacking | Uses victim's authenticated session to issue SOAP requests as the user | High |

### Lateral Movement
*No lateral movement techniques observed/reported in this advisory — the campaign is a single-hop, browser-scoped email exfiltration.*

### Collection
| ID | Technique | Use in campaign | Confidence |
|---|---|---|---|
| [T1114](https://attack.mitre.org/versions/v19/techniques/T1114/) | Email Collection | Harvests up to 90 days of victim email | High |
| [T1114.002](https://attack.mitre.org/versions/v19/techniques/T1114/002/) | Email Collection: Remote Email Collection | Collected via ZCS API calls, not local mail store | High |
| [T1119](https://attack.mitre.org/versions/v19/techniques/T1119/) | Automated Collection | 12-stage automated collection sequence on payload execution | High |
| [T1560](https://attack.mitre.org/versions/v19/techniques/T1560/) | Archive Collected Data | Emails GZIP-compressed before exfiltration | High |

### Command and Control
*No distinct C2 channel reported beyond the exfiltration infrastructure itself (Flowerbed/Catcher) — treated below under Exfiltration.*

### Exfiltration
| ID | Technique | Use in campaign | Confidence |
|---|---|---|---|
| [T1074.002](https://attack.mitre.org/versions/v19/techniques/T1074/002/) | Data Staged: Remote Data Staging | Data staged on actor VPS prior to onward transfer | High |
| [T1048](https://attack.mitre.org/versions/v19/techniques/T1048/) | Exfiltration Over Alternative Protocol | Dual-channel exfil (DNS + HTTPS) | High |
| [T1048.002](https://attack.mitre.org/versions/v19/techniques/T1048/002/) | Exfil Over Asymmetric Encrypted Non-C2 Protocol | Bulk data (emails, attachments) sent over HTTPS with Let's Encrypt certs | High |
| [T1048.003](https://attack.mitre.org/versions/v19/techniques/T1048/003/) | Exfil Over Unencrypted Non-C2 Protocol | Smaller values (email addr, passwords, 2FA codes) sent via Base32-encoded DNS A-record queries | High |

### Impact
*None reported — campaign is espionage/data-theft only, no destructive or availability-impacting behavior.*

---

## Indicators of Compromise

### CVE
- **CVE-2025-66376** — CWE-79 (stored XSS via unsanitized CSS `@import` in ZCS webmail). Patched in ZCS 10.1.13 and 10.0.18.

### Flowerbed C2/exfil infrastructure (domain — IP — first/last seen)
| Domain | IP | First seen | Last seen |
|---|---|---|---|
| zmailanalytics[.]com | 216.252.238[.]104 | 2025-07-08 | 2025-10-15 |
| zimbra-metadata[.]com | 216.252.238[.]18 | 2025-08-20 | 2025-10-14 |
| analyticemailmeter[.]com | 37.120.247[.]228 | 2025-09-24 | 2026-03-18 |
| emailanalytics.com[.]ua | 185.86.79[.]95 | 2025-09-24 | 2026-03-18 |
| mailnalysis[.]com | 104.248.134[.]194 | 2025-11-11 | 2026-02-17 |
| zimbrastat[.]com | 64.226.124[.]190 | 2025-12-18 | 2026-03-18 |
| zimbrasoft.com[.]ua | 193.238.152[.]66 | 2026-01-20 | 2026-03-18 |
| synacorzimbra[.]nl | 216.252.238[.]64 | 2026-02-03 | 2026-03-30 |
| istc-cloud[.]com | 194.156.103[.]193 | 2026-02-05 | 2026-03-30 |

### TLS certificate SHA-1 hashes (Let's Encrypt, per associated domain)
- zmailanalytics[.]com — `2e4f314bc9943cab5005d6fde0b271c74d47bc9d`
- *.i.zmailanalytics[.]com — `50a87d926621dd06389ba50d86e0ff574ed713a8`
- *.i.zimbra-metadata[.]com — `c5a72420e7bb308d078e62128430897f82194c95`
- *.i.analyticemailmeter[.]com — `8959c4d29e29f02ea94ea8bb21c8df2594c5549d`
- *.i.emailanalytics.com[.]ua — `62eb76432597694edb01c1fe57aab0cfe03a7178`
- *.i.mailnalysis[.]com — `cddf5c3be1e07f28140aed165b929bf2d614922a`
- *.i.zimbrastat[.]com — `18b3ad442ce73cc8656d51d75bbd7c855f2cb7e8`
- *.i.zimbrasoft.com[.]ua — `1b25041ececf2457eef0270fc1d785cec8ec9ded`
- *.i.synacorzimbra[.]nl — `e4fe6466a4f9a4249fe330651e914e45bbdca44a`
- *.i.istc-cloud[.]com — `b6b77c9a455225d525834a403ca9ef5481ed0447`

### Email addresses — infrastructure procurement
- ivanka.zurabishvili@proton[.]me
- zmul1@buildandconsulting[.]com
- garrysmithme@pinmx[.]net
- hostingclient@pinmx[.]net

### Email addresses / domains — phishing distribution
- c.laurent.ejfa@proton[.]me
- j.moreau.epsc@proton[.]me
- liberty.insights@proton[.]me
- addresses ending `@isofts.kiev[.]ua` (presumed-compromised)
- addresses ending `@navs.edu[.]ua` (presumed-compromised)

### File hashes (SHA-256) — malicious email samples
- `98df604ecc57f884a2e6ce3266a0013ad64455cac48442c2312cfa4765007aaf`
- `60db9abae75cd8ccc49dd7ea5feb41677566dcd442f12ebc5745ffd2810fb874`
- `b1f5beb1175fc5c7d1806a2f0d900eb124c54f0286c5c52b66eea7a6633adb1d`
- `1517b3caa495f6c4e832df9c75fc94667e3c233773f7fa4e056d5e30e5ead760`

### File hash (SHA-256) — Catcher's static `pixel.gif` response
- `ef1955ae757c8b966c83248350331bd3a30f658ced11f387f8ebf05ab3368629`

### File paths (host-based artifacts)
- `/opt/zimbra/log/mailbox.log` (ZCS SOAP request logging — victim-side)
- `/root/hits/tmp`, `/root/hits/ready` (Catcher staging dirs — attacker infra)

### Other artifacts
- Application Passcode named **"ZimbraWeb"** — anomalous, ZCS natively supports 2FA and has no legitimate reason to create one with this name.
- `localStorage` keys of the form `zd_comp_YYYY-MM-DD` left on victim endpoints, marking days of email already exfiltrated.
- Full STIX feeds: [AA26-204A.stix.xml](https://www.cisa.gov/sites/default/files/2026-07/AA26-204A.stix_.xml) / [AA26-204A.stix.json](https://www.cisa.gov/sites/default/files/2026-07/AA26-204A.stix_.json)

---

## Simulation Plan (Atomic Red Team)

**Caveat:** this campaign's core techniques (the CVE-2025-66376 SVG-smuggled XSS
payload, its 12-stage SOAP-abuse collection logic, and the Flowerbed/Catcher
exfil stack) are bespoke and have **no dedicated Atomic Red Team tests** — they
aren't generic enough to be in the public library. The tests below approximate
detection coverage for the *general* ATT&CK techniques the campaign maps to,
not the specific exploit chain. Validate exact test IDs against the current
[atomics-red-team](https://github.com/redcanaryco/atomic-red-team) repo before
running, since content changes over time.

**High confidence + likely atomic coverage** (prioritize these):
- **T1027.010 / T1027.013** — Command/file obfuscation: atomics exist for
  Base64-encoded and XOR-obfuscated payload execution (PowerShell/bash
  variants).
- **T1560** — Archive Collected Data: multiple published atomics (zip/tar via
  CLI, PowerShell `Compress-Archive`) — good proxy for the GZIP-archive
  exfil-staging step.
- **T1048.003** — Exfiltration Over Unencrypted Non-C2 Protocol: DNS-exfil
  atomics exist (e.g., chunked data via `nslookup`/PowerShell DNS queries) —
  good proxy for the Base32 DNS-exfil channel.
- **T1048.002** — Exfiltration Over Asymmetric Encrypted Non-C2 Protocol:
  atomics exist for HTTPS POST exfil.
- **T1087** — Account Discovery: atomics exist (`net user`, `Get-ADUser`,
  Azure AD/Entra user enumeration) — reasonable proxy for GAL enumeration
  behavior, though the real technique is API-based brute force, not
  local-host enumeration.
- **T1098 / T1556.006** — Account Manipulation / MFA tampering: atomics exist
  for adding auth methods, disabling MFA enforcement, creating app passwords
  in Azure AD/O365 — directly relevant since the victim environment here is
  a hosted webmail platform.
- **T1550.004** — Use Alternate Authentication Material (pass-the-cookie):
  atomics exist and are a good test for the AiTM/session-replay TTP used in
  this actor's prior campaigns.
- **T1595** — Active Scanning: atomics exist (e.g., Nmap-based scans) for the
  reconnaissance stage.

**Techniques with little or no atomic coverage** (not simulatable via the
standard library — consider a custom/tabletop exercise instead):
- T1587.001, T1587.004, T1588.002, T1588.007, T1608 — malware/exploit/tooling
  development and staging; these describe attacker-side capability building,
  not host-observable execution.
- T1583, T1583.003 — infrastructure acquisition; not a host/network-emulable
  action.
- T1589.001, T1589.002, T1593, T1596.005, T1597, T1597.002 — OSINT/recon
  against public or commercial data sources; outside endpoint/cloud telemetry
  scope.
- T1027.017 (SVG smuggling) — an emerging sub-technique; check current atomics
  coverage, likely absent as of this writing.
- T1203 (client-side exploitation via a webmail-specific XSS) — atomics exist
  for *some* client-execution scenarios (e.g., Office macros) but nothing
  approximates a browser-DOM XSS-in-webmail chain; recommend a custom
  detection engineering exercise using a lab ZCS instance instead.
- T1114 / T1114.002 (bulk email collection via API) — a few atomics touch
  mailbox export via `Get-MessageTrace`/Graph API; coverage is thin and worth
  validating directly.

**Priority order for simulation** (high confidence + atomic available first):
1. T1027.010 / T1027.013 — payload obfuscation
2. T1048.003 / T1048.002 — DNS + HTTPS exfiltration
3. T1560 — archive before exfil
4. T1098 / T1556.006 — MFA/app-passcode account manipulation (cloud/webmail focus)
5. T1087 — directory/GAL-style account discovery
6. T1550.004 — session cookie reuse
7. T1595 — reconnaissance scanning

---

## References
- Advisory: [CISA AA26-204A](https://www.cisa.gov/news-events/cybersecurity-advisories/aa26-204a)
- [MITRE ATT&CK Enterprise Matrix v19](https://attack.mitre.org/versions/v19/matrices/enterprise/)
- [MITRE D3FEND v1.4.0](https://d3fend.mitre.org/)
- AIVD/MIVD original LAUNDRY BEAR advisory (May 2025)
- Microsoft: Void Blizzard (May 2025)
- Unit 42: Russian Global Webmail Espionage (2026)
- Proofpoint: TA488 Targets Zimbra Mailservers with Half-Click Exploits (2026)
- Seqrite: Operation GhostMail (2026)
