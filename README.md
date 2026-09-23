<div align="right">

English | [简体中文](README.zh-CN.md)

</div>

# dsh-plugin-upgrade

**A headless-capable SOP skill for upgrading [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) (dsh) plugins after a dsh upgrade.**

dsh moves fast through 0.x releases and breaks plugin seams constantly. This skill turns "my plugin broke after the upgrade" from an archaeology dig into a repeatable procedure: evidence first, seam re-verification against the installed source, incremental fixes with per-fix commits, a verification ladder, and a version-sync tag.

Field-verified on the dsh-plugin-job-panel upgrade `0.1.5-rc.2 → 0.1.7-alpha.2` (2026-09-23): a rebuilt popover moved the React key from the row `<li>` to a component fiber, which this procedure located in minutes from installed-source evidence.

**Re-runs are idempotent.** Every completed upgrade writes a marker — `META/dsh-upgrade.json` (the dsh version it was verified against, the plugin commit, the seam slugs, which verification rungs were reached). The next run checks the marker first: same dsh version and unchanged code → a minutes-long quick re-verify instead of a full seam scan; drifted or missing → the full procedure, and the marker gets rewritten. Markers gate the scan, never the verify.

[![License: MIT](https://img.shields.io/badge/license-MIT-green.svg)](./LICENSE)
![verified](https://img.shields.io/badge/verified-dsh%200.1.7--alpha.2-blue)

## The SOP at a glance

| Step | What happens |
|---|---|
| §0 Principles | Evidence before edits; installed source = single truth; compatible fixes; small commits; never restart `dsh web` from inside it |
| §0.5 Headless mode | Every user question answered by a pinned recommended default; outward actions (publish/push/restart) prepared but never auto-executed |
| §1 Scope | Auto-inventory of installed third-party plugins and DSH-related skills; marker triage partitions candidates into skip vs work before any scan |
| §2 Source & versions | Locate the installed dsh, upstream tags, clone the repo if missing; side quest: re-verify the stale `dsh-plugin-dev-notes` skill itself |
| §4 Per-plugin | Marker fast path first (FAST → quick re-verify / WIDE → all seams / FULL → everything), then evidence: diagnostics files (which half broke?), verified baseline, diff only the touched dsh packages (`npm pack` old vs new), grep-verdict every seam |
| §5 Report | Failure scope / root cause with old-vs-new evidence / fix / how to verify |
| §6 Wrap-up | Verification ladder (tests → bundle smoke → cookie-probed routes → click-test), per-fix commits, the `META/dsh-upgrade.json` marker, strongly-recommended `verified-dsh/vX.Y.Z` tag |

The full procedure lives in [SKILL.md](SKILL.md) — that file is the skill; this README is only the front door.

## Headless mode

Every question the SOP asks has a recommended answer, so the skill can run unattended: say "无头" / "headless" / "不要问我" and each decision is auto-made and recorded for the final report. Red lines that never run autonomously: `npm publish`, `git push`, restarting `dsh web` — those are prepared as exact commands and handed to you. See SKILL.md §0.5 for the decision table.

## Install

Prerequisite: none (the skill is a single markdown file).

```sh
# via the skills CLI
npx -y skills add cholf5/dsh-plugin-upgrade
# or manually
git clone https://github.com/cholf5/dsh-plugin-upgrade.git ~/.agents/skills/dsh-plugin-upgrade
```

Works best alongside [dsh-plugin-dev-notes](https://github.com/cholf5/dsh-plugin-dev-notes) (the dual-face plugin development field notes this SOP inherits its verify-against-installed-source methodology from).

## Usage

Load the skill in your agent, then either:

- name the plugins to upgrade — the SOP starts immediately, or
- load it bare — it inventories your profiles and asks what to upgrade, or
- add headless wording — it runs the whole procedure on the recommended defaults.

Pair it with the companion [dsh-plugin-dev-notes](https://github.com/cholf5/dsh-plugin-dev-notes) skill, whose CHECKLIST.md this SOP reuses when the field notes themselves drift behind the target dsh version.

## License

[MIT](./LICENSE)
