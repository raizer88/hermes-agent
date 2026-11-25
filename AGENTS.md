# Hermes Agent - Development Guide

This root file is a hub: what applies everywhere, then a routing table. Each area's `AGENTS.md`
loads automatically when you work in that directory; read it before editing there.
`python scripts/check` caps this file at 12k chars and every root-to-area chain at 30k: long form
goes in `website/docs/developer-guide/`.

**Never give up on the right solution.**

## What Hermes Is

Hermes is a personal AI agent that runs one agent core across a CLI, a TUI, an Electron desktop
app, a web dashboard, ACP editors and a messaging gateway (25+ platform plugins). It is extended
through **plugins and skills**, not by growing the core.

Two invariants shape almost every design decision and are the lens for reviewing any change:

- **Per-conversation prompt caching is sacred.** Mutating past context, swapping toolsets,
  reloading memories or rebuilding the system prompt mid-conversation breaks the cached prefix and
  multiplies the user's cost; the ONE exception is context compression. Slash commands that change
  system-prompt state defer to the next session, with an opt-in `--now` (`/skills install --now`).
- **The core is a narrow waist; capability lives at the edges.** Every model tool is sent on
  every API call, so the bar for a new *core* tool is high. New capability should arrive as a
  CLI command + skill, a service-gated tool, or a plugin — not as core surface.

A change to the stock system prompt or `tools[]` fails `tests/ci/test_prompt_surface.py` until
`scripts/ci/prompt_surface.py render` is committed with it, and that snapshot
(`tests/fixtures/prompt_surface/`) needs a hermes-agent-core approval to merge.

## Contribution rubric

The project's intent layer, for contributors and for the triage sweeper (which may only close on
`implemented_on_main`, `cannot_reproduce` or `incoherent`; taste-based closes are a maintainer's
call, and when in doubt a PR stays open). Long form with examples:
`website/docs/developer-guide/contributing.md` § Contribution rubric.

**Wanted:** real bug fixes (repro on `main`, the exact line, the whole class incl. sibling paths);
reach at the edges (adapters, providers, models, UI features) wired into the existing setup UX;
god-file → module refactors; extending before duplicating (3+ PRs in one category → an ABC +
orchestrator); behaviour-contract tests; E2E with real imports against a temp `HERMES_HOME` for
resolution, config, security and I/O changes; salvage by cherry-pick so authorship survives.

**Rejected even when well-built:** hooks with no concrete consumer; new `HERMES_*` env vars for
non-secret config (`.env` is secrets only, behaviour goes in `config.yaml`); a new core tool when
terminal + file or a skill already does the job; `offset`/`limit` pagination on instructional tools;
"fixes" that destroy the feature they secure; outbound telemetry without an opt-in gate;
change-detector tests; plugins that touch core files; third-party products in the core tree (ship a
standalone plugin repo).

**Before you call it a bug,** verify the claim AND the intent (`git log -p -S "<symbol>"`): the
isolation is often the design (profiles are islands on purpose), an absence can be load-bearing,
and a fix that cannot point to the line where the bug manifests has an unverified premise.

**Security:** `SECURITY.md` is the scope authority. A §3.1 finding goes private (GitHub Security
Advisories or security@nousresearch.com), never into a public issue, PR, commit or comment; §3.2
hardening is ordinary public work. Name the §2 boundary crossed, with a repro on `main`.

**Footprint ladder** (take the highest rung that solves it): extend existing code → CLI command +
skill → service-gated tool (`check_fn` answers reachability/opt-in, never per-session surface) →
plugin → MCP server in the catalog → new core tool (fundamental, broadly useful, unreachable
otherwise).

## Development Environment

`source ./activate` (`activate.fish`, `activate.ps1`) provisions and activates the PM environment; pick an isolated `HERMES_HOME`/`HERMES_RUNTIME_DIR` first
(`website/docs/reference/package-management.md#developer-workflow`). Tests need the separate test
environment in `CONTRIBUTING.md`. **`python scripts/check`** (`just check`) runs every blocking
lint check CI runs, with CI's pinned tools; `--install-hook pre-push` runs it on every push (re-run
after pulling to refresh the hook). `just` lists every developer command (lefthook calls the same
recipes); a new command is a `scripts/` file plus a recipe. The `HX0xx`/`PS-P0x` rules and their
fixes are in `scripts/code_health/config.py`; `# noqa` does not waive a finding,
`# health: allow <RULE> -- <why>` does.

## Project Structure

The filesystem is canonical. Load-bearing entry points:

```
hermes-agent/
├── run_agent.py          # AIAgent facade; the turn loop lives in agent/turn_*.py
├── model_tools.py        # Tool orchestration, handle_function_call(); discovery in tools/registry.py
├── toolsets.py           # TOOLSETS dict, _HERMES_CORE_TOOLS
├── cli.py                # HermesCLI (REPL, slash dispatch) + hermes_cli/cli_*_mixin.py
├── hermes_state.py       # SessionDB facade; hermes_state_*.py siblings
├── hermes_constants.py   # get_hermes_home(), display_hermes_home() — profile-aware paths
├── scripts/              # check, run_tests.sh, code_health/, ci/
├── evals/                # live-model A/B and probe harnesses
└── website/              # Docusaurus docs (developer-guide/ holds the long-form area docs)
```

**User state:** `~/.hermes/config.yaml` (settings), `.env` (secrets only), `logs/` (`hermes logs`);
all profile-aware via `get_hermes_home()`.

### Packages and Python shape

A topic that outgrows its file becomes a folder: `<stem>/__init__.py` is the **facade** (its
docstring names each module's job, `__all__` is the surface, PEP 562 `__getattr__` only for a
measured startup cost or import hazard it names; no behaviour; `tools/connectors/` is the model),
`<stem>/<topic>.py` holds the code. Flat `<stem>_<topic>.py` siblings (`gateway/run.py`,
`tools/mcp_tool.py`) stay until touched; add none unless an area AGENTS.md still prescribes one
(`tui_gateway` RPC methods, `tools/` backends). A family folder keeps its ABC/Protocol, one module
per variant and a name → variant table (`tools/environments/`, its table still in
`terminal_tool_backends.py`). Tool folders register from `tool.py`; tests mirror folders.

- **Find code by topic:** `grep -rn "def name" <dir>/<stem>/ <dir>/<stem>_*.py`, not the facade.
- **Imports go at module top:** topic modules import one another, never the facade; a cycle means
  a shared name belongs in a lower module. Patch where the name is looked up.
- **Size and complexity are ratcheted per unit:** new functions CC ≤ 20, ≤ 300 lines, nesting ≤ 6;
  files ≤ 2,000 lines; units already over may only go down (move code into `<stem>/`).
- **Shape:** code imports on 3.11 (no PEP 695 syntax); import has no side effect but tool registration
  (no I/O, threads, config or env reads); input is parsed where it enters into a frozen dataclass, `Enum`,
  `Literal` or the area's wire model (Pydantic in `tui_gateway`); one return shape per function;
  dispatch by table, not `if name ==` ladders; collaborators are objects, not mixins; no module
  constant kept only for tests to patch.
- **No re-export shims for internal moves;** internal paths are not API. Moving a symbol means
  fixing its docs in the same PR (grep `website/docs`, `skills/`, every `AGENTS.md`).

## Rules that apply everywhere

- **No defensive clutter:** no wrappers or `try/except: pass` around code that cannot fail, no flags
  nobody sets, no dead code wired in without E2E proof. Comments keep the WHY, cut the WHAT.
- **Never infer process identity from argv substrings;** use
  `gateway.status.looks_like_gateway_command_line` / `hermes_cli.update_cmd._hermes_holder_subcommand`
  (HX003).
- **Never hardcode `~/.hermes`:** `get_hermes_home()` for paths, `display_hermes_home()` for text
  (HX001). `_get_profiles_root()` is HOME-anchored on purpose.
- **One process serves many profiles.** Code that runs outside a turn (boot probes, eviction,
  tickers, deferred callbacks, RPC methods, thread hops, child spawns) binds the owning profile scope
  explicitly; `os.environ`, module globals and import-time values hold the launch profile's, so an
  unbound read is a silent default-profile leak. Binding points and seams: `gateway/AGENTS.md`
  § Profile scope (HX002/HX004/HX005/HX012, PS-P05/P06).
- **Machine facts and executable lookup go through `hermes_platform`.**
- **Dependencies carry upper bounds** (`>=floor,<next_major`; git URLs and Actions pinned to a SHA);
  after editing `pyproject.toml` run `hermes pm lock` and commit `uv.lock`; never mutate a Hermes
  environment with raw pip/uv.
- **TypeScript (desktop, TUI):** feature-owned nanostores over threaded state, thin route roots,
  narrow hooks and colocated action modules, `interface` for props and shared shapes, table-driven
  dispatch over condition ladders; desktop keeps routes in `src/app`, atoms in `src/store`, pure
  helpers in `src/lib`.
- **Don't add core/plugin crossings:** core never imports a specific bundled plugin, and plugins
  use only the published plugin API (`hermes_cli/data/plugin_api.json`, `ctx`). Existing crossings
  are baselined; `scripts/boundary_lint.py` fails the test suite (not `scripts/check`) on new ones.
- **A new config key or slash command ships with its docs** (`tests/ci/test_docs_drift.py`).
- **Commits and PRs:** rebase onto `main` before merging (a squash from a stale branch silently
  reverts newer fixes); 1–2 invariant tests per fix, proven red on the base.
- **Tests:** always `scripts/run_tests.sh`, never bare `pytest`; behaviour contracts, never
  change-detectors or source-reading tests; host-specific behaviour is tested on that host with
  `@pytest.mark.platforms(...)`, never by faking `sys.platform`; tests never write to `~/.hermes/`.

## Routing Table

| Area | Read | Covers |
|---|---|---|
| `run_agent.py`, `agent/` | `agent/AGENTS.md` | turn phases, message-flow invariants, compression, model/aux resolution, i18n |
| `cli.py`, `hermes_cli/` | `hermes_cli/AGENTS.md` | CLI mixins, slash registry, config system, skins, `hermes update`, profiles / multiplex |
| `gateway/` | `gateway/AGENTS.md` | adapters, message guards, streaming, notifications, token locks, § Profile scope |
| `plugins/platforms/` new adapter | `gateway/platforms/ADDING_A_PLATFORM.md` | step-by-step adapter guide |
| `tools/`, `toolsets.py`, `model_tools.py` | `tools/AGENTS.md` | adding tools, registry, toolsets, delegation, session-scoped surface tools |
| `plugins/`, `hermes_cli/plugins*.py` | `plugins/AGENTS.md` | plugin kinds, native compat contract, in-tree policy, boundary lint |
| `plugin-catalog/` | `plugin-catalog/README.md` | catalog admission rules (mirrored in the developer guide; keep identical) |
| `tui_gateway/`, `ui-tui/` | `tui_gateway/AGENTS.md` | process model, JSON-RPC transport, slash flow |
| `web/`, `hermes_cli/web_routers/` | `web/AGENTS.md` | dashboard embeds the real TUI |
| `apps/desktop/` | `apps/desktop/AGENTS.md`, `apps/desktop/src/AGENTS.md` | state authority, gateway backend contract, slash palette, Bot Mode |
| `skills/`, `optional-skills/`, `agent/curator*.py` | `skills/AGENTS.md` | frontmatter, authoring standards, curator |
| `cron/`, kanban | `cron/AGENTS.md` | scheduler invariants, job fields, kanban dispatcher |
| `tests/` | `tests/AGENTS.md` | runner, placement, OS markers, `wine2e`, banned test shapes |
| `pm/`, `pyproject.toml` | `pm/AGENTS.md` | pinning policy, PM-owned environments, release-age rule |
| `hermes_platform/` | `hermes_platform/AGENTS.md` | host facts, resolvers |
| `acp_adapter/` | `website/docs/developer-guide/acp-internals.md` | ACP server for editors |
| `providers/` | `providers/README.md` | `ProviderProfile` registry and ABC |

PR workflow: `CONTRIBUTING.md` (maintainers: the `hermes-agent-dev` skill, not shipped here).

---

## This fork: building the remote-only desktop client

Everything above describes upstream. This repository is a personal fork whose only
build product is the desktop **`light`** (remote-only) variant, from branch
`remote-only`, based on upstream **v0.23.2** (`ae2a81ccc0`). Nothing below applies
to upstream's release lanes: no payload, no installer, no signed artifact.

### What the build produces

`python scripts/bundles/desktop.py --commit <sha> --variant light --clean` writes
the portable folder `apps/desktop/release/win-unpacked`: an Electron app with no
Python payload that can talk only to a remote Hermes gateway.
`resources/install-stamp.json` must read `payload: "light"`,
`source: "commit-build"`, `dirty: false`.

### Host prerequisites (Windows)

- Python 3.11–3.14 on `PATH`.
- Node **26+**. With fnm the binary is under
  `%APPDATA%\fnm\node-versions\v26.13.0\installation` — put that folder first on
  `PATH` (`EBADENGINE` otherwise, because `.npmrc` sets `engine-strict=true`).
- **pnpm 12.15.0 exactly** (`pm/lock.json`). Either the native `pnpm.exe` or the
  npm distribution, whose `pnpm.cmd` runs `node_modules/pnpm/bin/pnpm.mjs`.
  If a native pnpm starts but dies oddly (thread stack overflow, store lock
  "Accesso negato"), that is a host problem, not a repo problem: force the Node
  CLI with `HERMES_PNPM_CLI=1`. The driver also downloads its own pinned Node and
  pnpm into `.cache/desktop-inputs`; that is expected.
- ~10 GB free for the checkout, `node_modules`, the Electron download and the
  packaged app.
- Network for the clone, the pnpm store and Electron. Where schannel fails
  (`SEC_E_NO_CREDENTIALS`), use `git -c http.sslBackend=openssl`.

### Build

From a checkout of this branch:

```powershell
git clone https://github.com/raizer88/hermes-agent.git
cd hermes-agent
git checkout remote-only
pnpm install --frozen-lockfile
python scripts/bundles/desktop.py --commit (git rev-parse HEAD) --variant light --clean
```

`scripts\build-bundle.ps1 -Variant light` does the same from inside the repo. The
author's `rebuild\rebuild-light.ps1` (preflight with `-DryRun`, scratch clone,
`win-unpacked.new`) is a companion file kept next to the built app, **not in this
repository**.

`--commit` wants the **full 40-character** SHA of the checkout's `HEAD`, and the
tree must be clean: the driver re-checks both while building. Never use `npm`
(upstream refuses it) and never run `hermes update` here: it resets the tree to
`origin/main` and drops every fork commit.

PowerShell scripts must stay **ASCII-only**: Windows PowerShell 5.1 reads
BOM-less files as ANSI, so one em dash breaks parsing. If the execution policy
blocks `.ps1`, call it as
`powershell -NoProfile -ExecutionPolicy Bypass -File <script>`.

### Install the result

A running `hermes-light-*.exe` locks `resources\app.asar`: close every such
process first. Move the **folder**, never the exe alone:

```powershell
Remove-Item .\win-unpacked -Recurse -Force
Move-Item .\win-unpacked.new .\win-unpacked
```

The app's userData is per commit (`%APPDATA%\HermesLightCommit<sha>`). Copy
`connection.json`/`connections.json` and the sign-in state from the previous
build, or the user has to sign in again. Then repoint
`HKCU\Software\Classes\hermes\shell\open\command` and any Desktop or Start Menu
shortcut at the new exe.

### Verify before reporting success

- `resources/install-stamp.json`: `payload: "light"`, `dirty: false`, `commit` =
  the SHA you built, `baseVersion` = the upstream release this fork is based on
  (currently `0.23.2`) — **never `0.0.0`**.
- `resources/` contains no Python payload.
- `%LOCALAPPDATA%\hermes\logs\desktop.log` shows
  `[boot] Remote Hermes backend is ready` and
  `[updates] remote-only: latest release is vX (this build is <baseVersion>)`.
- No new `python.exe`, and the profile rail shows no "This device".
- Launch the exe with `ELECTRON_RUN_AS_NODE` **unset**; with it set, Electron runs
  as plain Node and the process exits silently.

### Traps already paid for

- **Integrity labels.** On a host whose workspace is sandboxed, files can carry a
  *Low* mandatory label. The exe then starts at low integrity and dies with
  `0x80000003` (crashpad "not connected"). That is not a build defect: repair the
  folder with `icacls .\win-unpacked /setintegritylevel "(OI)(CI)M" /T /C /Q`.
- `createLocalGatewayDials()` must stay inert: `main.ts` builds the registry at
  module load, so a throw there kills the process before the window exists.
- `bootstrapFailure` is global and latched: on `light` a refusal must not assign
  it, or the primary remote connection goes down too.
- An unsigned MSIX cannot be installed; light builds use `--win --dir`.
- `git apply --3way` leaves the patches uncommitted and `HEAD` on the tag: use
  `git am --3way`, since the patches are `format-patch` output.
- `project.version` in `pyproject.toml` is what a `--commit` build reports
  (`scripts/releases/commit_build.py::version_at`). Number it from the upstream
  base and amend it on every rebase: at `0.0.0` every release looks newer and the
  About banner never turns off.

Report the commit you built, the stamp fields and the log lines, plus anything you
could not verify. Do not claim "built and working" without the stamp and the boot
log.
