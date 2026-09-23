---
name: dsh-plugin-upgrade
description: "Step-by-step SOP for upgrading DeepSeek Harness (dsh) plugins after a dsh upgrade: inventory, seam re-verification against installed source, incremental fixes with per-fix commits, live verification, and version-sync tagging. Load when a plugin broke after a dsh upgrade, or the user asks to adapt/upgrade plugins (or DSH-related skills) to a new dsh version."
---

# dsh Plugin Upgrade SOP

A repeatable procedure for making dsh plugins work again after dsh itself
moves (breaking changes are frequent across 0.x releases). Field-verified on
the dsh-plugin-job-panel upgrade `0.1.5-rc.2 → 0.1.7-alpha.2` (2026-09-23).

## 0. Principles (read first)

1. **Evidence before edits.** Diagnose from artifacts (diagnostics files,
   dump-config, timestamps, installed source) before touching code. One
   session located a total client-half failure in minutes this way.
2. **Installed dsh source = the single source of truth.** Package READMEs,
   built `lib/*.js`, and `lib/types/*.d.ts` under the running install beat
   docs, blogs, and memory. Reference repos drift.
3. **Compatible fixes.** Where cheap, make detection tolerate both the old
   and the new shape — plugins then survive adjacent versions.
4. **Small commits.** One logical fix = one commit (conventional messages).
   Never pile the whole upgrade into one commit.
5. **Never restart `dsh web` from inside dsh web.** The agent session itself
   runs under it; a restart kills the conversation. Hand restarts to the
   user, or use a throwaway `DSH_HOME` + separate port for experiments.
6. The dsh web restart empties the job registry; bundle installs/removals do
   not hot-reload (restart needed); `lib/client.js` edits hot-swap via HMR;
   patch rows hot-reload.

## 0.5 Headless mode

Activate when the user asks for it in any wording ("无头", "headless",
"自动", "不要问我", "自己决定"), or when the upgrade runs as an autonomous
goal with no human in the loop. In headless mode every question this SOP
would ask is answered with the recommended default below, and each
substitution is recorded for the final report — the user must be able to
reconstruct every decision from the report alone.

| Question (SOP ref) | Headless default |
|---|---|
| Which plugins to upgrade (§1) | Every **locally checked-out** third-party plugin in the profile that fails seam verification. Registry-only third-party plugins get `dsh plugin update <pkg> -w` (the author's fix) and a report if still broken — never hand-patch code you don't maintain. |
| Target version (§3) | The **installed, running** dsh version — fix what actually runs; note newer upstream tags in the report. |
| Fix approach (§4.4) | Client half first, both-shape-compatible detection, one regression test per discovered shape. |
| Version-sync tag (§6) | Yes — prefixed `verified-dsh/vX.Y.Z` + `META/dsh-baseline.txt` + README badge. |
| npm publish / git push (§6) | **Never executed autonomously.** Prepare the exact commands, report them. |
| Restart `dsh web` (§6) | **Never self-initiated.** Hand the instruction to the user in the report. |

Everything else — code fixes, tests, rebuilds, commits, the tag — proceeds
without asking: it is all local and commit-revertible. For multi-round runs,
wrap the objective with `create_goal` so the upgrade continues across rounds;
if a blocking condition outlives the round budget, mark the goal `blocked`
with the concrete condition instead of guessing past it.

## 1. Scope

- User names the plugins to upgrade → start immediately.
- Bare load → ask. Offer the actual inventory:
  ```sh
  cat ~/.dsh/profiles/*/package.json | python3 -c "import json,sys; [print(b) for p in sys.stdin if (p:=json.loads(p)) for b in p.get('dsh',{}).get('profile',{}).get('bundles',[])]"
  ls ~/.agents/skills/   # DSH-related skills can also go stale — offer them too
  ```
  Third-party plugins in the bundles list (`dsh-*`, `@*/dsh-*`) are the
  candidates; official `@deepseek-ai/*` entries are not.

## 2. Locate dsh source and versions

1. Installed version (this is what runs):
   ```sh
   dsh --version 2>/dev/null || npx @deepseek-ai/dsh --version
   node -e "console.log(require(require('child_process').execSync('npm root -g').toString().trim()+'/@deepseek-ai/dsh/package.json').version)"
   # npx-only installs: ~/.npm/_npx/*/node_modules/@deepseek-ai/dsh/package.json
   ```
2. Upstream tags:
   ```sh
   git ls-remote --tags https://github.com/deepseek-ai/deepseek-harness.git | tail
   npm view @deepseek-ai/dsh version
   ```
3. Full source checkout (needed for history/diff): search common locations
   (`~/dev/**/deepseek-harness`) first; if absent `git clone --tags
   https://github.com/deepseek-ai/deepseek-harness.git ~/dev/dsh/deepseek-harness`.
4. **Skill-staleness side quest:** if the `dsh-plugin-dev-notes` skill's
   baseline (its CHECKLIST.md recheck log) is older than the target version,
   run that CHECKLIST.md procedure once this session and update the skill
   (fix drifted facts, append a log row). The skill itself is a victim of the
   same drift.

## 3. Confirm the target

Report to the user: installed version, newest official tag, plugin(s) in
scope, and whether the dev-notes skill needs a recheck. Ask: upgrade to
`vX.Y.Z` (usually = installed version) — yes/no.

## 4. Per-plugin upgrade

### 4.0 Evidence first (before reading plugin code)

```sh
cat ~/.dsh/storages/<plugin>.json 2>/dev/null   # activation diagnostics, if the plugin writes them:
   # services/wraps/routes fields → host half health; lastRouteHitAt vs lastJobTrackedAt
   # → which half is dead; updatedAt vs file mtime → local-UTC offset for the timeline
dsh --profile web --dump-config | grep -A3 "<plugin-id>"   # is the Loader row still composed?
ls -la ~/.dsh/profiles/*/node_modules/<plugin>             # link: install still resolves?
```

### 4.1 Determine the plugin's verified baseline

Check, in order: README "verified against dsh@..." line and version badge;
the plugin's own **Version-sensitive seams table** (the
dsh-plugin-dev-notes README convention — it IS the upgrade checklist); git
tags and commit messages mentioning a dsh version; `package.json` notes.

### 4.2 Baseline known → diff the dependency surface

Do **not** diff the whole dsh repo. Enumerate the dsh packages the plugin
actually touches (its imports; `dsh.client.inject`; host `inject` services;
the seams table rows), then per package:

```sh
mkdir -p /tmp/dsh-upgrade && cd /tmp/dsh-upgrade
npm pack @deepseek-ai/<pkg>@<old> @deepseek-ai/<pkg>@<new>
tar -xzf *-<old>.tgz -C old-<pkg> --strip-components=1; tar -xzf *-<new>.tgz -C new-<pkg> --strip-components=1
diff -u old-<pkg>/lib new-<pkg>/lib | head -100
# plus history: git -C ~/dev/dsh/deepseek-harness log v<old>..v<new> -- packages/<that-pkg>
```

Grep-verdict each seam in the seams table against the **new** installed
source (not the tarball): DOM selectors, React fiber shapes, service names
(`ctx.reflect.provide(...)`), slot seat names, route APIs, static module
table (`grep -o "function AS(){return{...}"` in
`dsh-web-frontend/dist/assets/index-*.js`).

### 4.3 Baseline unknown → hard-verify current code

Verify every fact the plugin assumes about dsh against installed source;
anything unverifiable or contradicted is a fix candidate. Prioritize by
surface: client half (DOM/fiber/module-table — drifts most) → host half
(services/routes — drifts less) → manifest (`dsh.client`, patch row).

### 4.4 Fix discipline

- Prefer client-half fixes: they hot-swap into the running browser (HMR);
  host-half fixes need a `dsh web` restart — schedule with the user.
- Build/verify: `npm run build`, `node --check lib/*.js`, `npm test`
  (node --test, dependency-free). Add a regression test pinning each new
  shape discovered (model fibers/DOM as plain objects — no jsdom needed).
- Update, in the same change: the plugin's seams table, the verified badge
  and "verified against" line (both README languages if present), and the
  recheck log if the plugin keeps one.
- Commit per fix. Rebuild `lib/client.js` and commit it too (it ships).

## 5. Report, then iterate

Tell the user: failure scope (which half, which flows), root cause (what
changed in dsh, with the old-vs-new evidence), how it was fixed, and how to
verify. Ask what else to change; loop back to 4.x per item.

## 6. Verification ladder and wrap-up

Verify cheap → expensive, and record what was verified how:

1. `node --check` + full `npm test`.
2. Bundle factory smoke in `vm` with a stubbed `window.__ModuleLoader__`.
3. Live host probe — **a cookie is mandatory**; an unauthenticated curl
   returns 401 for every `/api` path (the fence rejects before routing), so
   401 proves nothing: mint one from the launch URL
   (`curl -s -c /tmp/dsh-cookies.txt "http://127.0.0.1:3080/?token=<token>" -o /dev/null`),
   then expect the handler's body vs 404 "not found".
4. Client behavior: user click-test after a page refresh (HMR usually
   hot-swapped the fix already); read the diagnostics file for objective
   confirmation (e.g. a fresh `lastRouteHitAt`).

Wrap-up:

- Working tree committed (per-fix commits already made).
- **Version-sync tag — recommend it strongly.** Two options, ask the user:
  (a) tag the plugin with the dsh version it is verified against — note the
  namespace collides with the plugin's own semver tags; (b) preferred:
  prefixed tag `verified-dsh/vX.Y.Z` plus a `META/dsh-baseline.txt` file
  (what dsh-plugin-dev-notes does) plus the README badge.
- If the plugin is on npm: `npm publish` (2FA/OTP or granular token caveats
  apply), then `npm view <pkg> version`.
- Remind the user of the restart/refresh actually needed, and clean up
  `/tmp/dsh-upgrade`.

## 7. Pitfalls (field-tested)

| Pitfall | Reality |
|---|---|
| Trusting "it 401s, so the route is gone" | Fence rejects before routing — cookie probe only |
| Diagnosing from the plugin's happy-path docs | Read the diagnostics file / dump-config first |
| Diffing the whole dsh repo | Diff only the touched packages, old-vs-new tarballs |
| Fixing to the new shape only | Tolerate both shapes; the next upgrade will thank you |
| Client fix committed but lib/ not rebuilt | `lib/client.js` ships built; `npm run build` before commit |
| Restarting dsh web mid-session | Kills the agent's own conversation — hand it to the user |
| Assuming the dev-notes skill is current | Its baseline may predate the target — run its CHECKLIST.md |
| One giant commit | Per-fix commits; the tag then points at a reviewable diff |
