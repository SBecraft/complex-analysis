---
name: investigate
description: Correlate Windows endpoint and Azure AD cloud logs to investigate a potential compromise, anchored on an identity, hostname, or IP
arguments:
  - name: anchor
    description: The identity (username/UPN), hostname, or IP address to start the investigation from
    required: true
---

# Multi-Source Investigation

## Step 1: Orient

Read `logs/README.md` for the environment/host/IP table and the
correlation-key + timestamp-normalization rules. These differ across
sources (Windows Security, Sysmon, Azure AD sign-in, Azure AD audit) per
CLAUDE.md's guidance — do not assume a generic "logs" abstraction.

Normalize `$anchor` to all forms it might appear as: on-prem `DOMAIN\user`
vs. cloud `user@domain` UPN, bare hostname vs. FQDN.

## Step 2: Cloud timeline skeleton

Dispatch the `cloud-analyst` subagent against `logs/cloud/` (Azure AD
sign-in + audit logs) for `$anchor`. Have it build a timeline: normalize
timestamps, diff `ipAddress` / `location` / `deviceDetail` across sign-in
entries to surface anomalies (unfamiliar location, missing/unmanaged
device, risk flags), and flag any persistence-relevant audit events
(MFA/security-info registration, OAuth consent grants, role or group
changes) tied to the same identity.

## Step 3: Endpoint evidence

Dispatch the `endpoint-analyst` subagent against `logs/windows/` (Windows
Security + Sysmon logs) for `$anchor` and for any hostname tied to it via
the sign-in logs' `deviceDetail`. Have it search:
- **Before** the cloud anomaly's timestamp, for initial-access/execution
  evidence (suspicious process chains, encoded commands, LOLBins).
- **After** the cloud anomaly's timestamp, for lateral-movement and
  persistence evidence (logons to other hosts, new-account creation,
  suspicious process creation).

## Step 4: Correlate

Merge both subagents' findings into a single cross-source timeline,
normalized to UTC per `logs/README.md`'s rules. Join records using the
documented correlation keys:
- Identity (bare account name, case-insensitive, domain/UPN stripped)
- Hostname (FQDN vs. short name)
- IP address — note whether it's an **external** IP (cloud-facing pivot) or
  **internal** IP (on-prem lateral-movement pivot); these often play
  different roles in the same incident.

## Step 5: Expand scope

If step 3 or 4 surfaces a new host or account not covered by the original
`$anchor` (e.g. a newly created local account, a second compromised
workstation), repeat steps 2-4 using it as a fresh anchor, until no new
pivots emerge.

## Step 6: Output

Write a structured markdown report to
`analysis/correlation-[date]-[subject].md`:
- Frontmatter: anchor identity/host/IP, date range covered, sources consulted
- **Timeline**: consolidated, UTC-normalized, chronological
- **Initial access & execution**: what happened on the endpoint
- **Cloud persistence**: MFA/OAuth/role changes tied to the compromise
- **Lateral movement**: additional hosts/accounts affected
- **IOCs**: IPs, usernames, hostnames, hashes, process names — grouped by role (attacker infrastructure vs. affected internal assets)
- **ATT&CK techniques**: mapped with evidence and confidence (high/medium/low)
- **Recommended containment**: account/credential resets, token revocation, OAuth consent removal, host isolation, backdoor account removal
- **Gaps**: anything that couldn't be confirmed from available sources
