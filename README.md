# polinrider-guard

Detect, auto-heal, and monitor for the **PolinRider** supply-chain worm across your GitHub repositories — **free-plan friendly** (works on private repos with no paid features; GitHub Actions minutes are free).

PolinRider (DPRK-linked, aka part of the "Contagious Interview" cluster) infects developer machines, steals GitHub credentials, and injects payloads into config and source files across every repo the victim can push to, rewriting history and force-pushing so the change looks old. A file committed *inside* a repo can't stop it (it pushes with `--no-verify` and can delete the file), so every layer here runs **on GitHub's side**.

> ⚠️ This cleans and protects **repositories**. It does not disinfect a developer's machine — see [Machine cleanup](#machine-cleanup). Until each machine is clean, guard/monitor will keep auto-healing (repos stay safe, but you'll see cleanup commits).

## Indicators of compromise

| Where | What |
|---|---|
| Any file | marker strings `A8-4892`, `RS260605`, `M260630A`, `__inzCR`, `rmcej%otb%`, or a hardcoded C2 IP |
| Config files | `postcss/tailwind/eslint/next/vite/babel/jest/prisma/ecosystem.config.*` with real code, a long whitespace run, then a hidden payload (normal ~100–300 B, infected 7–50 KB) |
| npm | `npm/lib/cli.js` ≈1.6 MB (normal ≈215 B) |
| VS Code / Cursor | `main.inz.cjs` beside `resources/app/out/main.js`; `__inzCR` header on `main.js` |
| Electron apps | payload appended to `resources/app/main.js` after `//# sourceMappingURL` |
| Dropped files | `temp_auto_push.bat`, `config.bat`, fake `*.woff2` (real WOFF2 starts with bytes `wOF2`), `.vscode/tasks.json` with `runOn: folderOpen` |
| Source files | payloads also appear in ordinary `routes/*.js`-style source — scan every file, not just configs |

## The three layers

### 1. Per-repo guard — `repo-files/.github/`
Copy `polinrider-guard.yml` (workflow) and `polinrider-scan.sh` (scanner) into a repo. On every push / PR / force-push, and on a 6-hour schedule, GitHub Actions:
- scans the whole tree for the indicators;
- **auto-heals** — strips the payload and pushes a clean commit; if the push was a force-push, restores the branch;
- logs, in a `polinrider-guard` issue, the **account that pushed** (recorded by GitHub, cannot be faked) vs the **name on the commit** — a mismatch = a forged commit — plus the force-push flag and the pre-push SHA.

Run the scanner locally too: `bash .github/polinrider-scan.sh` (or `--clean` to strip).

### 2. Rulesets — `tools/apply-ruleset.ps1` *(public repos, or private on a paid plan)*
Blocks force-pushes + deletions on all branches and requires the guard check on protected branches — rejecting the malicious push outright. **On GitHub Free + private repos this API returns 403** (rulesets aren't available there); rely on layers 1 & 3, which need only free Actions.

### 3. Central monitor — `monitor-repo/`
A separate private repo nobody pushes code to. Hourly, it scans **every branch of every repo** you can access via the API, strips/reverts anything infected, and re-adds the guard workflow to any repo that lost it. Findings land in issues here — out of the malware's reach. This covers the gap layer 1 leaves (a push to a stale branch that doesn't yet carry the workflow).

## Setup

1. **Lock down your account first:** rotate password, enable 2FA, revoke **all** OAuth apps, PATs, and SSH keys. (2FA does **not** protect `git push` — it uses tokens/SSH, which bypass it.)
2. **Deploy the guard** to your repos: `node tools/deploy-guard-api.mjs --apply` (add `--repos list.txt` to scope it).
3. **Clean existing infection:** `node tools/clean-remote.mjs --apply` (add `--deep` to include source files). **Strip-only — it never rewrites history.**
4. **Stand up the monitor:** `node tools/setup-monitor.mjs`, then add a `GH_PAT` repo secret (a classic token with `repo`+`workflow`; it lives only inside GitHub Secrets).
5. Optional, where supported: `tools/apply-ruleset.ps1`.

Tokens for the scripts: a classic PAT with `repo` + `workflow`. Never commit it; keep it out of the repo.

## The one rule that matters most

**Never `git reset` / roll a branch back to "clean" it.** PolinRider *amends* real commits, so genuine work sits on top of the infection — a reset destroys it. Only strip the payload from the offending file and commit forward. The scanner's `--clean` does exactly this.

## Machine cleanup

Guard/monitor keep repos clean but can't disinfect a PC. On each machine:
- Reinstall Node if `npm/lib/cli.js` is bloated; reinstall VS Code/Cursor if a `main.inz.cjs` exists.
- Delete `temp_auto_push.bat` / `config.bat` and fake `*.woff2`.
- Disable VS Code's `task.allowAutomaticTasks`.
- Run a full AV scan; rotate every secret that lived on the machine.

## Contents

```
repo-files/.github/     guard workflow + scanner  (drop into each repo)
monitor-repo/           central monitor workflow + guard-files
tools/                  deploy, clean, scan, monitor-setup, ruleset scripts
```

## Disclaimer

Provided as-is, no warranty. Review scripts before running them against your repos; test on one repo first. Indicators reflect the variants we observed and may drift as the campaign evolves.

## License

MIT.
