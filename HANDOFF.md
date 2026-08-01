# HANDOFF

Session notes for picking this project back up. Newest entries at top.

## 2026-08-01 — Root README now describes the project, not the dataset

**What happened:**
- Found `README.md` at the repo root was a **byte-identical copy** of
  `logs/README.md`, with `logs/README.md` itself deleted in the working tree
  (both changes uncommitted, carried in from before this session). Net effect:
  the top-level README documented only the synthetic dataset, and the path
  that CLAUDE.md and `.claude/commands/investigate.md` Step 1 both tell readers
  to open for correlation keys and timestamp rules no longer existed.
- Restored `logs/README.md` (`git checkout --`) rather than repointing those
  two references, keeping the dataset writeup where the repo's own docs say it
  lives.
- Rewrote root `README.md` as an actual project README: purpose and the two
  workflows, the four log sources and why they aren't abstracted, directory
  layout, `/ingest-ti` and `/investigate` (behavior, output paths, known
  limits) with links to the example reports in `analysis/`, the two analyst
  subagents and why correlation lives in the command rather than in them, the
  dataset pointer, and the WSL/nvm + `xmllint` environment notes.
- Committed as `5ddd4a9` on `master` — `README.md` only; restoring
  `logs/README.md` made the tree match HEAD again, so it needed no commit.

**Not yet done:**
- `analysis/investigation-summary.md` (dated 2026-07-25) is still **untracked**
  and overlaps heavily with `analysis/correlation-2026-07-10-mrodriguez-compromise.md`.
  Left out of the commit and unmentioned in the README — needs a decision on
  whether it's a keeper, a duplicate to delete, or something to fold into the
  correlation report.
- Everything from the 2026-07-24 entry carries over unchanged: no GitHub
  remote, `/investigate` never run directly as a slash command, no committed
  dataset-generation script, no timestamp-normalization helper, and the
  analysts still have only `Read`/`Bash`.

## 2026-07-25 — Renamed investigation report naming convention to "correlation"

**What happened:**
- Renamed `analysis/investigation-2026-07-10-mrodriguez-compromise.md` to
  `analysis/correlation-2026-07-10-mrodriguez-compromise.md` via `git mv`
  (`02b52d8`), then updated the three places that documented or produced the
  old `investigation-[date]-[subject].md` naming — CLAUDE.md's
  `/investigate` description, `.claude/commands/investigate.md`'s Step 6
  output path, and this file's 2026-07-24 entry — to `git commit`
  (`b7ec5c6`) the new `correlation-[date]-[subject].md` convention
  consistently across docs, command, and the one existing report.
- No behavioral change to the workflow itself — purely a naming-convention
  update so future `/investigate` runs and the existing sample report agree.

**Not yet done:** (carried over, unchanged this session — see 2026-07-24 entry)

## 2026-07-24 — Ran both analysts, correlated findings, git-initialized the repo

**What happened:**
- Dispatched `endpoint-analyst` against `logs/windows/` and `cloud-analyst`
  against `logs/cloud/` (the real registered subagent types — the harness
  picked them up this session; last session had to use `general-purpose`
  stand-ins with the instructions pasted inline because the types weren't
  recognized yet). Saved their raw output to `analysis/endpoint.md` and
  `analysis/cloud.md`.
- Both agents independently converged on the same incident without shared
  context — `mrodriguez` / `185.220.101.47` — validating that the synthetic
  dataset's signal is coherent and discoverable per-source, not just when
  read as a whole.
- Manually correlated the two write-ups into
  `analysis/correlation-2026-07-10-mrodriguez-compromise.md` (renamed
  2026-07-25 — see that date's entry below — from
  `investigation-2026-07-10-mrodriguez-compromise.md`): timeline
  alignment, user/IP correlation, full attack chain, confidence assessment,
  and gaps. Key finding: the endpoint analyst's "~7-hour dead zone" between
  the C2 callback (09:08) and the FS01 lateral movement (16:04) is exactly
  where the cloud analyst's sign-in/MFA/OAuth-consent activity (11:00-11:51)
  sits — neither analysis could see this alone; only correlating both did.
  This is effectively the `/investigate` methodology validated end-to-end,
  even though the slash command itself wasn't invoked directly (done as
  discrete agent dispatches + manual correlation instead).
- Initialized git: `git init` (default branch `master`, not renamed to
  `main`), added `.gitignore` (excludes `.claude/settings.local.json` — local
  permission overrides, not shared config — plus standard Python/OS cruft),
  set repo-scoped (not `--global`) git identity, and made the initial commit
  (`9e241d4`, 24 files). No GitHub remote yet — user plans to create it
  later.

**Not yet done:**
- No GitHub remote configured — repo is local-only.
- `/investigate` still hasn't been invoked directly as a slash command (the
  workflow it encodes has now been manually validated, but not the command
  itself end-to-end).
- The synthetic-data generation script still isn't committed (scratchpad-only
  from last session) — regenerating/extending the dataset means rewriting it.
- No automated timestamp-normalization helper exists — still documented-only
  in `logs/README.md`.
- `endpoint-analyst`/`cloud-analyst` still only have `Read`/`Bash` tools (no
  `Grep`/`Glob`) — both ran fine against this dataset's size, but worth
  revisiting if the dataset grows.

## 2026-07-23 — Synthetic multi-source dataset + /investigate command + analyst subagents

**What happened:**
- Built out the "multi-source investigation" half of CLAUDE.md's purpose, which had no data or tooling until now. Planned first (see
  `<home>/.claude/plans/i-m-investigating-a-potential-dynamic-stearns.md`
  for the full design rationale), then implemented after the user confirmed
  scope (full haystack-with-needles realism, plus scaffold `/investigate` now
  rather than later).
- Added two subagents first, at the user's direction, before the dataset existed:
  `.claude/agents/endpoint-analyst.md` (Windows Security + Sysmon) and
  `.claude/agents/cloud-analyst.md` (Azure AD sign-in + audit). Note their
  `tools:` frontmatter only lists `Read`/`Bash` — no `Grep`/`Glob` — worth
  revisiting if they struggle to locate/filter records efficiently against
  larger log files.
- Generated the synthetic dataset under `logs/windows/` (Sysmon +
  Windows Security, native multi-`<Event>` XML) and `logs/cloud/` (Azure AD
  sign-in + audit, JSONL) — **not** the plan's originally-proposed 4-way
  split (`logs/sysmon/`, `logs/windows-security/`, `logs/azure-ad-signin/`,
  `logs/azure-ad-audit/`) — the folder names were changed to `windows/`/`cloud/`
  to match the paths the user actually referenced when invoking the two
  analyst subagents. 3 days (2026-07-09/10/11), incident contained entirely
  in 07-10, ~137 Sysmon events / ~168 Security events / ~257 sign-ins / ~62
  audit entries, malicious signal is 1-3% of each file. Generated via a
  one-off Python script (not committed — lived in the session scratchpad)
  rather than hand-authored, since hand-writing ~600+ internally-consistent
  events reliably isn't practical.
- Scenario builds directly on `sysmon-parser`'s existing fictional malicious
  sample (`samples/event3.xml` — `mrodriguez`/`WKSTN-017`/`Invoice_7742.docm`)
  rather than inventing a disconnected cast, extending it into a full
  hybrid-identity compromise: phishing → C2 callback → risky Azure AD
  sign-in → MFA/OAuth persistence → on-prem lateral movement to `FS01` →
  `helpdesk2` backdoor account. Full details and correlation-key/timestamp-
  normalization documentation live in `logs/README.md`.
- Added `.claude/commands/investigate.md`, mirroring `ingest-ti.md`'s
  frontmatter/step structure, which dispatches `cloud-analyst` then
  `endpoint-analyst` and correlates their output into an `analysis/` report.
- Verified structural validity (Python's `xml.etree`/`json` stdlib parsers —
  `xmllint` isn't installed in this environment) and cross-source IOC
  consistency (shared `ProcessGuid`, both attacker IPs, `helpdesk2`) before
  handing off.

**Not yet done:**
- The generation script itself isn't in the repo (scratchpad-only) — if the
  dataset needs to be regenerated or extended (e.g. more days, more noise),
  it'll need to be rewritten rather than rerun.
- `/investigate` hasn't been run end-to-end yet.
- No automated timestamp-normalization helper exists yet — `logs/README.md`
  documents the three formats and the rule, but nothing enforces/implements
  it in code.

## 2026-07-23 — First end-to-end /ingest-ti run

**What happened:**
- Ran `/ingest-ti` end-to-end for the first time, against CISA advisory
  [AA26-204A](https://www.cisa.gov/news-events/cybersecurity-advisories/aa26-204a)
  (LAUNDRY BEAR / Void Blizzard Zimbra ZCS campaign, CVE-2025-66376).
  `analysis/` directory created; report written to
  `analysis/ti-2026-07-23-laundry-bear-zimbra-zcs.md`.
- `defuddle parse ... --markdown` worked cleanly for this CISA URL. Note the
  nvm gotcha from the prior session still applies each new shell — had to
  manually `. "$NVM_DIR/nvm.sh" && nvm use node` since `source ~/.bashrc`
  alone didn't put `node`/`defuddle` on PATH in this session's shell.
- Two other candidate URLs failed before landing on the CISA one, worth
  remembering as failure modes:
  - `https://threatfox.abuse.ch/` and `https://research.checkpoint.com/intelligence-reports/`
    are landing/index pages, not reports — `defuddle` either returns
    marketing boilerplate or errors with "No content could be extracted."
  - `https://research.checkpoint.com/2026/20th-july-threat-intelligence-report/`
    (a real report URL) is gated behind an **AWS WAF JS challenge**
    (`x-amzn-waf-action: challenge`, HTTP 202 empty body) — both `defuddle`
    and `WebFetch` get empty content. No workaround attempted (would mean
    evading a bot-protection control); Check Point reports are effectively
    unreachable by this command as currently built.

**Not yet done:**
- No MITRE ATT&CK/Atomic Red Team data source is wired in — technique
  mapping and atomic-test suggestions still rely on the model's own
  knowledge, not a pinned local dataset. The Atomic Red Team section for
  this report leans heavily on caveats for exactly this reason (couldn't
  confirm exact atomic test IDs).
- Command still fails silently-ish on JS-walled or index/landing-page URLs;
  might be worth adding an explicit pre-check or clearer error guidance to
  the command doc.

## 2026-07-23 — First slash command: /ingest-ti

**What happened:**
- Added `.claude/commands/ingest-ti.md`, the first workflow command for this project — implements "threat intel processing" (see CLAUDE.md Purpose): takes a report URL, extracts content, maps behaviors to MITRE ATT&CK TTPs, pulls IOCs, suggests Atomic Red Team tests, and writes a report to `analysis/ti-[date]-[campaign-name].md`.
- The pasted draft called `defuddle-cli` with `defuddle "$url" --format markdown` — that's the old deprecated CLI's syntax. Corrected to the real `defuddle` package's subcommand form: `defuddle parse "$url" --markdown`.

**Not yet done:**
- `/ingest-ti` hasn't been run end-to-end yet — untested against a real URL.
- `analysis/` output directory doesn't exist yet.
- No MITRE ATT&CK or Atomic Red Team data source is wired in — the command currently relies on the model's own knowledge for technique mapping and atomic test suggestions, not a lookup against a local/pinned dataset. Worth revisiting if accuracy/currency matters (ATT&CK framework updates periodically).

## 2026-07-23 — Environment setup: Node/npm fixed, defuddle installed

**State when starting:** Repo was empty. Only work this session was environment tooling, not project code.

**What happened:**
- Installed `defuddle-cli` globally via the Windows-native npm (`/mnt/c/Program Files/nodejs`), which this WSL shell was borrowing. npm flagged it as deprecated (merged into `defuddle`).
- Swapped to the `defuddle` package, but hit `node: not found` — the Windows npm global shims require a bare `node` on PATH, which WSL doesn't provide (only `node.exe`, and Windows `node.exe` also mishandles WSL-style paths like `/mnt/c/...` during module resolution, producing bogus UNC paths).
- Root-caused this as "no native Linux Node in this WSL install" rather than patching the shim further.
- Installed **nvm** (user-local, no sudo) and Node v24.18.0 LTS natively under it.
- Reinstalled `defuddle` (0.19.2) under the nvm-managed npm — confirmed working (`defuddle --help`, `defuddle --version`).
- Old Windows-global installs (`@angular/cli`, old `defuddle`) still exist under `C:\Users\<username>\AppData\Roaming\npm` — untouched, not currently used.

**Gotcha for next session:** Any *already-open* shell from before nvm was installed won't have it loaded (bashrc changes only apply to new shells). If `defuddle`/`node` resolve to the `AppData/Roaming/npm` path again, run `source ~/.bashrc`.

**Not yet done:**
- No project code, ingestion scripts, or workflow scaffolding exists yet.
- `defuddle` was installed anticipating it'll be used to pull clean article/report text out of HTML threat intel sources during the "threat intel processing" workflow (see CLAUDE.md) — not yet wired into anything.
- If `@angular/cli` (or anything else under the old Windows npm) is needed from WSL, it should be reinstalled under nvm the same way.
