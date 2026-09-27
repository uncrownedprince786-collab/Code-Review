# Ponytail — security audit of `hooks/` and `scripts/`

**Audited version:** v4.10.0 · commit `e3ba2aa6f1e6f0bc4d69eb09c9f0d0a93af56156` (2026-09-14)
**Date of audit:** 2026-09-28
**Upstream:** https://github.com/dietrichgebert/ponytail (MIT)
**Reason:** this repo is a personal backup / working copy. Ponytail installs hooks that
execute on the local machine on every session start and every prompt submit, so the
executable surface was read before installing.

## Scope

**Audited in full — every file, line by line:**

- `hooks/` — 12 files: `ponytail-activate.js`, `ponytail-config.js`, `ponytail-instructions.js`,
  `ponytail-mode-tracker.js`, `ponytail-runtime.js`, `ponytail-subagent.js`,
  `ponytail-statusline.ps1`, `ponytail-statusline.sh`, and the 4 hook-registration JSON
  files (`claude-codex-hooks.json`, `copilot-hooks.json`, `cursor-hooks.json`,
  `qoder-hooks.json`)
- `scripts/` — 6 files: `uninstall.js`, `cursor-hooks.js`, `build-openclaw-skills.js`,
  `check-rule-copies.js`, `check-versions.js`, `publish-openclaw-skills.js`
- `package.json` (install lifecycle and dependency surface)

**NOT audited** — out of scope for this pass. Do not read this file as clearing them:

- `skills/`, `commands/`, `AGENTS.md` — the prompt text itself (cannot harm the machine,
  but it is what changes agent behaviour; see "Non-security caveat")
- `.opencode/plugins/ponytail.mjs` — the npm `main` entry, executes under OpenCode
- `pi-extension/`, `ponytail-mcp/` — separate executables with their own package.json
- `tests/`, `benchmarks/`, and the per-agent adapter folders
  (`.cursor/`, `.windsurf/`, `.agents/`, `.clinerules/`, `.kiro/`, `.qoder*/`, and others)

## Verdict

No malicious code, no data exfiltration, no network activity, no telemetry.
Reasonable to install. The findings below are things to know, not reasons to refuse.

## What executes, and when

| Event | Script | Host |
|---|---|---|
| `SessionStart` | `ponytail-activate.js` | Claude Code, Codex, Copilot, Cursor |
| `SubagentStart` / `PreToolUse(task)` | `ponytail-subagent.js` | Claude Code, Qoder |
| `UserPromptSubmit` / `beforeSubmitPrompt` | `ponytail-mode-tracker.js` | all |
| statusline refresh | `ponytail-statusline.{ps1,sh}` | Claude Code (opt-in) |

All are `node <plugin>/hooks/*.js` with a 5-second timeout.

## Findings that count in favour

1. **No `postinstall` / `preinstall` script, and zero runtime dependencies.**
   `package.json` declares only a `test` script. This eliminates the single largest npm
   supply-chain risk class: there is no code that runs at install time, and no
   transitive dependency tree to trust.
2. **No network calls anywhere in the audited files.** Grepped for `http(s)://`,
   `fetch(`, and `require` of `http` / `https` / `net` / `dns` / `tls` — zero hits in any
   shipped code path. No telemetry, no phone-home, no update check.
3. **No `child_process` in anything that ships.** The only `spawnSync` is in
   `scripts/publish-openclaw-skills.js`, a maintainer-only release script that shells out
   to the `clawhub` CLI. It is excluded from the npm `files` array, so it is never
   installed. Same for `build-openclaw-skills.js`.
4. **No access to credentials.** The only environment variables read are its own
   (`PONYTAIL_DEFAULT_MODE`, `PONYTAIL_HIDE_STATUS`, `PONYTAIL_QUIET_STARTUP`,
   `PONYTAIL_SUBAGENT_MATCHER`), host-detection flags, and path variables
   (`CLAUDE_CONFIG_DIR`, `APPDATA`, `XDG_CONFIG_HOME`). No `*_KEY` / `*_TOKEN` /
   `*_SECRET` reads. No `eval`, no `new Function`.
5. **Hooks fail open and never block.** Every I/O path is wrapped in try/catch with
   silent failure. There are explicit anti-hang guards — a 1-second stdin fallback with
   `.unref()` — added for a real Windows freeze where Claude Code's PowerShell wrapper
   swallows piped stdin so `end` never fires (upstream #443).
6. **Uninstall is surgical, not blunt.** `uninstall.js` removes only ponytail's own
   statusline segment and preserves other plugins' (`caveman && ponytail` keeps caveman);
   `cursor-hooks.js` strips only entries matching `ponytail-*.js` and leaves every other
   Cursor hook intact. Both **refuse to write** when the target JSON is malformed,
   warning instead of overwriting.
7. **Path handling uses an allowlist, not escaping.** `isShellSafe()` permits only
   `[A-Za-z0-9 _.\-:/\\~]` before a path is embedded in any shell command, with a
   documented manual-setup fallback otherwise. This is the right way round.

## Findings to be aware of

### 1. Files written outside the project

Installing ponytail means accepting writes to these paths:

- `~/.claude/.ponytail-active` — current mode flag (or `~/.cursor/`, `~/.qoder/`,
  `$PLUGIN_DATA`, depending on host)
- `~/.claude/.ponytail-statusline-nudged` — marker so the setup offer appears only once
- `%APPDATA%\ponytail\config.json` (Windows) or `~/.config/ponytail/config.json` —
  written only by `/ponytail default <mode>`
- `~/.cursor/hooks.json` — only if you run `scripts/cursor-hooks.js install`
- `~/.claude/settings.json` — the `statusLine` key (see next finding)

### 2. The hook injects text that steers your agent to edit your global settings

`ponytail-activate.js` appends this to the context it feeds the model:

> `STATUSLINE SETUP NEEDED: ... To enable, add this to <settings.json>: ...`
> `Proactively offer to set this up for the user on first interaction.`

This is the plugin asking your agent to modify `~/.claude/settings.json` on its behalf.
It is benign, and it is the plugin's own documented feature — but structurally it is a
hook writing instructions into the model's context aimed at a config file. The
consequence is that a session can end with your global Claude Code settings changed. It
fires at most once, guarded by the nudge flag. To prevent it entirely, set
`PONYTAIL_HIDE_STATUS=1` or create the config with `hideStatus: true` before first run.

### 3. `-ExecutionPolicy Bypass` in the suggested Windows statusline command

The command it proposes is:

```
powershell -ExecutionPolicy Bypass -File "<plugin>/hooks/ponytail-statusline.ps1"
```

Standard practice for running an unsigned local script, and the script itself is 12
lines that read the flag file and print a coloured label — nothing more. But it does
place an execution-policy bypass into your global settings, and it runs on every
statusline refresh. Worth a conscious yes rather than a default yes.

### 4. Every prompt you submit is piped to a hook

`ponytail-mode-tracker.js` receives the full text of each prompt on stdin as JSON. It
lowercases it, regex-matches for `/ponytail ...` and the phrases `stop ponytail` and
`normal mode`, then discards the rest. **It does not persist or transmit prompt text.**
This is the most privacy-sensitive dataflow in the plugin and the one worth re-verifying
yourself after any upstream update — it is about 35 lines and quick to re-read.

### 5. `uninstall.js` reformats `~/.claude/settings.json`

It rewrites the whole file with `JSON.stringify(settings, null, 2)`, so existing
indentation and key order are replaced. Harmless — the file is plain JSON, so comments
were never valid there — but it produces a larger diff than the change warrants. Back
the file up before uninstalling if you keep it under version control.

### 6. Unvalidated regex from environment (theoretical only)

`ponytail-subagent.js` calls `new RegExp(process.env.PONYTAIL_SUBAGENT_MATCHER, 'i')`
with no complexity check, so catastrophic backtracking is possible. Only reachable by
setting the variable yourself, and bounded by the 5-second hook timeout. Not a real
concern; recorded for completeness.

## Non-security caveat

Two things a code audit does not and cannot cover:

- **The README's headline numbers** (`~54% less code`, `~20% cheaper`, `~27% faster`,
  `100% safe`) are the author's own benchmarks against their own baseline. The
  methodology is published and reproducible under `benchmarks/`, which is more than most
  projects offer — but they are claims to verify, not established facts. "100% safe" is
  an absolute that no benchmark can support.
- **The real risk here is behavioural, not technical.** The point of this tool is to make
  a coding agent build less. If that tuning is wrong for a given project, the failure
  mode is under-building, which is harder to notice than a crash. Trial it on something
  low-stakes first.

## Reproducing this audit

```bash
grep -rn -E "https?://|fetch\(|child_process|execSync|spawn|eval\(|new Function" hooks scripts
```

```bash
grep -n -E "writeFileSync|unlinkSync|rmSync|mkdirSync" hooks/*.js scripts/*.js
```

Re-run after any upstream refresh. These findings apply only to the commit named at the
top of this file.
