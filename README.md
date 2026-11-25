# Hermes Desktop — Remote-Only Fork

> **Disclaimer — read this first.**
> This is an unofficial, personal fork of
> [NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent).
> I am **not** affiliated with, endorsed by, sponsored by, or connected to
> Nous Research in any way. "Hermes", "Hermes Agent" and the Hermes name and
> marks belong to their respective owners; this fork claims **no rights** to
> them and grants none. Nothing in this repository is an official Nous Research
> release or build. For the official project, use the upstream repository.

## What this repository is

A fork that builds the Hermes **Desktop client** in upstream's `light`
variant: a desktop app that can talk **only to remote Hermes gateways**. It
ships no local runtime and it can never create one.

In practice this build:

- has no bundled Python payload, no local venv and no `install.ps1`;
- never starts a local gateway/backend;
- never adopts an existing Hermes installation found on the machine;
- lists only remote connections — "This device" is never shown;
- on first run offers only **Connect to existing Hermes**;
- cannot install updates, but can tell you when a newer official release exists.

The agent runs elsewhere (in my case on a Raspberry Pi gateway); the desktop is
just the UI.

## Changes from upstream

Base: upstream `v0.23.2` (`ae2a81ccc0`).
Branch: `remote-only`. All changes live in `apps/desktop/electron/` plus the
build driver; the core Python and `gateway/` are untouched.
Rebased onto each new upstream release: the patch set is additive and touches
only files upstream changes rarely.

| # | Change | Where |
|---|--------|-------|
| M1 | `installShape()` reports `'light'` instead of collapsing it to `'checkout'` | `install-stamp.ts` |
| M2 | `ensureRuntime()` refuses a local bootstrap on a light artifact, **without** latching the global `bootstrapFailure` (latching would take the remote connection down too) | `main.ts` |
| M3 | "This device" is never enumerated, dialed or shown: local routes refuse with a readable message, roster rows are dropped and the renderer never receives the local entry | `connection-registry.ts`, `main.ts` |
| M4 | First-run overlay offers only the existing-gateway route; no local install card, no install target, no bootstrap progress | `bootstrap-state.ts`, `desktop-install-overlay.tsx`, `main.ts` |
| M5 | `HERMES_DESKTOP_IGNORE_EXISTING` is always in effect on light: a pre-existing `%LOCALAPPDATA%\hermes` install is never adopted | `backend-resolution.ts` |
| M6 | Every local-gateway entry point fails fast before spawning `hermes gateway ...` or touching a local control socket | `local-gateway.ts` |
| M7 | A self-contained update **checker** (GitHub `/releases/latest` redirect, 12 h cache) plus an unsupported updater strategy: the app can see that a newer official client exists, but no download/stage/apply path can run. Shown as an About banner and a toast, both linking to the release page | `remote-only-update-check.ts`, `main.ts`, `preload.ts`, `store/updates.ts`, About UI |
| Build | Windows light builds target `--dir` (portable `win-unpacked`) instead of MSIX; `build-bundle.ps1` accepts `-Variant light` | `scripts/bundles/desktop.py`, `scripts/build-bundle.ps1` |

No forked core: upstream's `gateway/` package is the library underneath the
Python core and is left exactly as upstream ships it.

The built artifact declares itself in `resources/install-stamp.json`:

```json
{
  "commit": "9b1e875feeafa7f8262f15a863a097ade4571fac",
  "dirty": false,
  "payload": "light",
  "baseVersion": "0.23.2",
  "displayVersion": "0.23.2",
  "updateMechanism": "external"
}
```

## Build (Windows)

Prerequisites:

- Python **3.11–3.14** on `PATH`;
- Node **26** (the repo's `.npmrc` sets `engine-strict=true`);
- **pnpm 12.15.0** exactly (`pm/lock.json`; `npm` is refused by the root
  `preinstall` and `engines.npm`);
- a **clean** checkout — the driver refuses a dirty tree and later re-checks
  that `HEAD` is still the admitted commit.

```powershell
git clone https://github.com/raizer88/hermes-agent.git
cd hermes-agent
git checkout remote-only
pnpm install --frozen-lockfile
python scripts\bundles\desktop.py --commit (git rev-parse HEAD) --variant light --clean
```

Or, with the wrapper script:

```powershell
scripts\build-bundle.ps1 -Variant light
```

### Build it with an agent

If you work with an LLM coding agent, point it at [`AGENTS.md`](AGENTS.md): the
last section, *"This fork: building the remote-only desktop client"*, is a
self-contained runbook — prerequisites, the two build paths, how to swap the
folder without breaking a running app, the checks that prove the artifact is
genuinely remote-only, and the traps that have already cost someone an afternoon
(low-integrity files, `ELECTRON_RUN_AS_NODE`, native pnpm, `.ps1` encoding). Hand
it the line *"read AGENTS.md and build the remote-only client"* and it has
everything it needs.

The light variant skips the TUI/web payload assembly, which is why it is much
smaller than a bundled build (the packaged folder is still dominated by the
Electron runtime, ~426 MB). Output:

```
apps\desktop\release\win-unpacked\hermes-light-<short-sha>.exe
```

For the current build that is `hermes-light-9b1e875.exe` (about 235 MB).

It is **portable**: launch it from the folder. Nothing is installed and nothing
is signed, so Windows SmartScreen will warn on first launch. The build asks
electron-builder for `--dir`, not MSIX: an unsigned MSIX cannot be installed
without trusting a certificate.

> The driver validates the revision and a clean tree during the build. Commit
> your work before starting a build and do not edit the checkout while it runs.

## First run

1. Launch `hermes-light-<short-sha>.exe` from `release\win-unpacked\`.
2. Choose **Connect to existing Hermes**.
3. Enter the gateway base URL (for example `https://hermes.example.com`) and
   complete token or browser sign-in.
4. Manage connections later in **Settings → Gateways**.

Expected behaviour on a remote-only build:

- no `python.exe` appears in Task Manager;
- `%LOCALAPPDATA%\hermes\tools` and `%LOCALAPPDATA%\hermes\installs` are not
  created (or stay empty);
- no `gateway.pid` is written;
- with the remote gateway down, the app reports it unreachable and does not
  fall back to a local backend.

## Updates

The About page (and a one-per-tag toast) can show
*"X.Y.Z is available — this build is remote-only and does not update itself"*
with a button that opens the GitHub release page. There is deliberately no
"Update now": the app never downloads, stages or applies anything.

To move to a newer upstream version, rebase `remote-only` onto the desired
upstream commit and rebuild.

**Do not run `hermes update` on a checkout that carries these changes.** It
resets the tree to `origin/main` and removes them.

## Maintenance

- `origin` = upstream `NousResearch/hermes-agent`, `fork` = this repository.
- One branch (`remote-only`) with one commit per change (M1–M7 + build), so
  conflicts during a rebase stay localized.
- The client is only UI, so it can stay on an older version for a long time.

## License and name

Code inherited from upstream is distributed under the upstream license; see
[`LICENSE`](LICENSE). This fork adds no license of its own.

This repository is **not** affiliated with Nous Research and is **not** an
official Hermes distribution. No rights to the "Hermes" name, logo or other
marks are granted or implied. No warranty; use at your own risk.
