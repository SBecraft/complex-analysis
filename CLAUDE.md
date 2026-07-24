# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project status

This repository is currently empty — no code, dependencies, or build tooling exist yet. The sections below describe the intended purpose and scope of the project so that future work stays aligned. There are no commands to run (build/lint/test) until initial project scaffolding is created; update this file once tooling exists.

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

- `.claude/commands/ingest-ti.md` — `/ingest-ti <url>`. Implements the threat intel processing workflow: fetches a report URL via `defuddle parse <url> --markdown`, extracts TTPs (mapped to MITRE ATT&CK IDs) and IOCs, suggests Atomic Red Team tests per technique, and writes a structured report to `analysis/ti-[date]-[campaign-name].md`. The `analysis/` output directory does not exist yet — it's created on first run of this command.

## Environment notes

This is a WSL (Ubuntu) environment. Node.js is managed via **nvm**, not the Windows Node install:

- `nvm` lives at `~/.nvm`; `~/.bashrc` sources it automatically for new interactive shells.
- If `node`/`npm`/global CLI tools (e.g. `defuddle`) fail with `node: not found` or resolve to `/mnt/c/Users/.../AppData/Roaming/npm/...`, the shell hasn't loaded nvm — run `source ~/.bashrc` (or open a new terminal) rather than reinstalling anything.
- Do not fall back to the Windows-native `node.exe`/npm global installs under `AppData/Roaming/npm` — mixing them causes module path resolution errors (Windows-style path handling breaks on WSL paths like `/mnt/c/...`).
- Global CLI tools currently installed under nvm: `defuddle` (HTML/article content extraction — used for pulling clean text out of threat intel reports/pages during ingestion).
