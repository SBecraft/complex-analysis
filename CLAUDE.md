# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project status

This is a documentation/workflow repo, not an application — there is no build, dependency install, or test suite to run. It's a git repository (initialized locally; no GitHub remote configured yet). Both workflows described below have working slash commands, and the multi-source investigation workflow additionally has a synthetic practice dataset and two analyst subagents. See `HANDOFF.md` for session-by-session history.

## Purpose

This project builds repeatable workflows for complex security analysis tasks. Two workflows are in scope:

1. **Threat intel processing** — ingest threat intel reports, extract TTPs (tactics, techniques, and procedures), and produce simulation plans from them.
2. **Multi-source investigation** — correlate endpoint and cloud log sources to support security investigations.

## Log sources

Analysis workflows are expected to work across these log sources:

- Windows Security events
- Sysmon events
- Azure AD sign-in logs
- Azure AD audit logs

When building ingestion, parsing, or correlation logic, design with this specific set of sources in mind (field naming, event ID schemes, timestamp formats, etc. differ across them) rather than a generic "logs" abstraction.

## Slash commands

- `.claude/commands/ingest-ti.md` — `/ingest-ti <url>`. Implements the threat intel processing workflow: fetches a report URL via `defuddle parse <url> --markdown`, extracts TTPs (mapped to MITRE ATT&CK IDs) and IOCs, suggests Atomic Red Team tests per technique, and writes a structured report to `analysis/ti-[date]-[campaign-name].md`. Doesn't work against JS-walled pages (e.g. AWS WAF challenge pages) or index/landing pages — needs a direct URL to an actual report.
- `.claude/commands/investigate.md` — `/investigate <anchor>`. Implements the multi-source investigation workflow: takes an identity/hostname/IP anchor, dispatches the `cloud-analyst` subagent against `logs/cloud/` and the `endpoint-analyst` subagent against `logs/windows/`, correlates their findings (shared identities, IPs, timeline alignment), and writes a report to `analysis/correlation-[date]-[subject].md`.

## Subagents

- `.claude/agents/endpoint-analyst.md` — analyzes Windows Security + Sysmon logs (authentication events, process execution, persistence, lateral movement indicators). Tools: `Read`, `Bash`.
- `.claude/agents/cloud-analyst.md` — analyzes Azure AD sign-in + audit logs (authentication anomalies, privilege changes, resource access, cross-source correlation hints). Tools: `Read`, `Bash`.

## Practice dataset

`logs/` contains a synthetic (fictional, not real-incident) dataset spanning all four log sources above, for exercising the investigation workflow: `logs/windows/` (Sysmon + Windows Security XML) and `logs/cloud/` (Azure AD sign-in + audit JSONL), covering 2026-07-09 through 2026-07-11 with a compromise scenario embedded in mostly-benign noise. Full scenario writeup, environment/host/IP table, and correlation-key + timestamp-format documentation live in `logs/README.md` — read that before writing any code that parses or correlates these sources.

## Environment notes

This is a WSL (Ubuntu) environment. Node.js is managed via **nvm**, not the Windows Node install:

- `nvm` lives at `~/.nvm`; `~/.bashrc` sources it automatically for new interactive shells.
- If `node`/`npm`/global CLI tools (e.g. `defuddle`) fail with `node: not found` or resolve to `/mnt/c/Users/.../AppData/Roaming/npm/...`, the shell hasn't loaded nvm — run `source ~/.bashrc` (or open a new terminal) rather than reinstalling anything.
- Do not fall back to the Windows-native `node.exe`/npm global installs under `AppData/Roaming/npm` — mixing them causes module path resolution errors (Windows-style path handling breaks on WSL paths like `/mnt/c/...`).
- Global CLI tools currently installed under nvm: `defuddle` (HTML/article content extraction — used for pulling clean text out of threat intel reports/pages during ingestion).
- `xmllint` is **not** installed in this environment. To validate the XML log files under `logs/windows/`, use Python's stdlib (`xml.etree.ElementTree`), not `xmllint`.
- Git identity is configured locally (repo-scoped, not `--global`) as SBecraft / 139415354+SBecraft@users.noreply.github.com. No GitHub remote yet.
