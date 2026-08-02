# complex-analysis

This repo is my working space for Module 8 — Complex Analysis in AI Cyber Defense Ops, a Just Hacking Training (JHT) course offered by Women in CyberSecurity (WiCyS) and taught by Anton Ovrutsky. All modules found in the ai-defense-labs repo at https://github.com/SBecraft/ai-defense-labs

The course centers on the Claude ecosystem — Claude Code and Claude Desktop — rather than treating AI as a single chat model bolted onto existing tools. Claude Code in particular behaves more like a productivity suite than a model: it ships with Hooks, Skills, Slash Commands, MCP support, and session/context management, and the course is built around learning how to compose those pieces into real detection engineering and threat intel workflows.

This module's project builds repeatable workflows for complex security analysis
tasks, implemented as Claude Code slash commands and subagents. It is a
documentation/workflow repo — there is no application, build, dependency
install, or test suite.

Two workflows are in scope:

1. **Threat intel processing** — ingest a threat intel report, extract TTPs
   (mapped to MITRE ATT&CK) and IOCs, and produce a simulation plan.
2. **Multi-source investigation** — correlate endpoint and cloud log sources
   into a single timeline to support a security investigation.

Both have working slash commands. The investigation workflow additionally ships
a synthetic practice dataset and two analyst subagents.

## Log sources

Everything here is designed against this specific set of sources — field
naming, event ID schemes, and timestamp formats genuinely differ between them,
so the workflows avoid a generic "logs" abstraction:

- Windows Security events
- Sysmon events
- Azure AD sign-in logs
- Azure AD audit logs

## Layout

```
.claude/
  commands/
    ingest-ti.md       /ingest-ti <url>
    investigate.md     /investigate <anchor>
  agents/
    endpoint-analyst.md   Windows Security + Sysmon
    cloud-analyst.md      Azure AD sign-in + audit
logs/                  synthetic practice dataset (see logs/README.md)
  windows/             Sysmon + Windows Security, native multi-<Event> XML
  cloud/               Azure AD sign-in + audit, JSONL
analysis/              generated reports (see below)
CLAUDE.md              guidance for Claude Code in this repo
HANDOFF.md             session-by-session history, newest first
```

## Workflows

### `/ingest-ti <url>` — threat intel processing

Fetches the report with `defuddle parse <url> --markdown`, extracts TTPs mapped
to MITRE ATT&CK technique IDs plus IOCs, suggests Atomic Red Team tests per
technique, and writes `analysis/ti-[date]-[campaign-name].md`.

Needs a direct URL to an actual report. It does not work against index/landing
pages (no extractable article content) or JS-walled pages such as AWS WAF
challenge pages. ATT&CK/Atomic Red Team mapping currently relies on model
knowledge — no pinned local dataset is wired in — so technique and atomic test
IDs should be spot-checked.

Example output: [`analysis/ti-2026-07-23-laundry-bear-zimbra-zcs.md`](analysis/ti-2026-07-23-laundry-bear-zimbra-zcs.md)
(CISA AA26-204A, LAUNDRY BEAR / Void Blizzard Zimbra ZCS campaign).

### `/investigate <anchor>` — multi-source investigation

Takes an identity, hostname, or IP as the anchor and normalizes it to every
form it might appear as (on-prem `DOMAIN\user` vs. cloud `user@domain`, bare
hostname vs. FQDN). Dispatches `cloud-analyst` against `logs/cloud/` and
`endpoint-analyst` against `logs/windows/`, correlates their findings on shared
identities, IPs, and timeline alignment, and writes
`analysis/correlation-[date]-[subject].md`.

Example output: [`analysis/correlation-2026-07-10-mrodriguez-compromise.md`](analysis/correlation-2026-07-10-mrodriguez-compromise.md),
with the two per-source analyses it was built from in
[`analysis/endpoint.md`](analysis/endpoint.md) and
[`analysis/cloud.md`](analysis/cloud.md).

[`analysis/investigation-summary.md`](analysis/investigation-summary.md)
compiles across both workflows — the three investigation documents above plus
the LAUNDRY BEAR TI report — into one write-up with a unified attack chain,
IOCs, ATT&CK mapping, confidence assessment, and containment steps. It also
carries the comparison none of the individual reports make: the TI advisory
resembles the incident in TTPs (durable mail-scoped persistence) but shares no
infrastructure with it, so it is context, not attribution.

## Subagents

- **`endpoint-analyst`** — Windows Security + Sysmon: authentication events,
  process execution, persistence, lateral movement. Tools: `Read`, `Bash`.
- **`cloud-analyst`** — Azure AD sign-in + audit: authentication anomalies,
  privilege changes, resource access, cross-source correlation hints. Tools:
  `Read`, `Bash`.

Each analyzes its own sources in isolation; correlation happens in
`/investigate`, not inside the agents. That separation is deliberate — it is
what makes the practice dataset a real test, since neither agent can see the
other half of the attack chain.

## Practice dataset

`logs/` holds a **synthetic** (fictional, not a real incident) dataset covering
all four log sources over 2026-07-09 through 2026-07-11, with a hybrid-identity
compromise embedded in mostly-benign noise (malicious events are 1–3% of each
file). The scenario runs phishing → C2 callback → risky Azure AD sign-in →
MFA/OAuth-consent persistence → on-prem lateral movement → backdoor account.

[`logs/README.md`](logs/README.md) has the full scenario writeup, the
environment/host/IP table, the correlation keys, and the timestamp-format
differences between sources. **Read it before writing any code that parses or
correlates these logs** — naive string sorting will not correctly interleave
Sysmon timestamps with the other two sources.

## Environment notes

WSL (Ubuntu). Node.js is managed via **nvm** at `~/.nvm`, not the Windows Node
install. If `node`/`npm`/`defuddle` fail with `node: not found` or resolve
under `AppData/Roaming/npm`, the shell hasn't loaded nvm — run
`source ~/.bashrc` or open a new terminal rather than reinstalling. Do not fall
back to Windows-native `node.exe`; mixing the two breaks module resolution on
WSL paths.

`xmllint` is not installed — validate the XML under `logs/windows/` with
Python's stdlib (`xml.etree.ElementTree`).

See [`CLAUDE.md`](CLAUDE.md) for the full working guidance and
[`HANDOFF.md`](HANDOFF.md) for session history.
