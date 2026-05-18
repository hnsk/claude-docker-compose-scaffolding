# claude-docker-compose-scaffolding

A generic, language-agnostic **seed** for AI-friendly, all-in-Docker-Compose
projects. It establishes one consistent style of development so an AI
agent gets structured, re-readable output instead of approving ad-hoc
commands and parsing truncated terminal scrollback.

It is a **starting point, not a dependency**: once a project is seeded it
is standalone and never re-clones this repo.

## What it ships

- **`devctl`** — a small Python CLI (console script in a project-local
  `.venv`) with thin `tools/scripts/*` shims:
  - `capture` — wrap any command; full stdout+stderr → a log file +
    `manifest.json` + `index.jsonl`. Prints `LOG=` first, `EXIT=` last;
    the wrapper itself exits 0. Read the `LOG=` path — never `| tail`.
  - `compose` — single-compose-file `docker compose` wrapper.
  - `run` / `logs` / `ps` — captured one-shot exec / logs / status.
  - `shell` — interactive shell into a running service (not captured).
  - `test` — captured test runner (`--full/--unit/--changed/--ci/
    --parallel`) in the long-lived `test`-profile service.
  - `show-tree` — dump a dir's files with banners (refuses paths outside
    the repo, skips binaries) instead of `find | xargs cat`.
  - `peek` — read-only partial view of one file (`--lines A:B`,
    `--head/--tail N`, `--match REGEX [--context N]`, `-n`) instead of
    raw `sed -n`/`head`/`tail`/`grep`; stdout only, repo-root-confined.
  - `clean` — wipe declared scratch dirs, bounded + captured.
  - Config via `tools/devctl.toml` (service names, test command, clean
    targets) — no CLI code edits per project. Zero third-party runtime
    deps (stdlib `tomllib`; needs Python ≥ 3.11 on the host).
- **Project skills** (`.claude/skills/dc-*`) wrapping the above.
- **Conventions** baked into a generated `CLAUDE.md`: everything in
  Docker Compose, latest-stable image rule, capture-everything,
  absolute paths, `./tmp/` scratch.
- **`TESTING.md`** — authoritative testing policy (always installed):
  all code tested, two-axis tagging from day one for selective +
  parallel runs, long-lived test DB reset by TRUNCATE/rollback,
  provider-agnostic `--ci` entrypoint. Stack-agnostic in the seed,
  adopted to the concrete stack at bootstrap.
- **TODO.md / NEXT.md epic workflow** — `TODO.md` authoritative; hard
  stop after each epic (commit → update TODO/NEXT → wait for the user).

## Prerequisites

- **Docker + Docker Compose** (all dev/test/run happens in containers).
- **Python ≥ 3.11** on the host (only `devctl` itself runs on the host;
  needs stdlib `tomllib`). No other host tooling.
- **Claude Code** with the two skills from this repo installed.

## One-time skill install

Make this repo's slash commands available from any project:

```
git clone https://github.com/hnsk/claude-docker-compose-scaffolding ~/src/dcs
ln -s ~/src/dcs/.claude/skills/docker-compose-scaffolding            ~/.claude/skills/
ln -s ~/src/dcs/.claude/skills/docker-compose-scaffolding-introduce  ~/.claude/skills/
```

(Symlink so a `git pull` in `~/src/dcs` refreshes both skills. Copying
the dirs works too — then re-copy to update.)

## Set up a new project — step by step

1. **Make the project dir** and `cd` into it (empty dir, or an existing
   repo you want to adopt the convention):
   ```
   mkdir ~/git/myproject && cd ~/git/myproject
   ```
2. **Write a plan file**, e.g. `PLAN.md`. This is the finalized brief the
   bootstrap consumes — its quality drives the generated epics. Include:
   - project name + one-paragraph summary;
   - the stack (languages/runtimes — bootstrap pins **latest stable**);
   - services needed (DB? cache? a test runner?);
   - the work breakdown (becomes `TODO.md` epics).

   Iterate it to "final" in Claude Code **plan mode** before step 3 —
   bootstrap does not re-plan, it only consumes.
3. **Bootstrap**:
   ```
   /docker-compose-scaffolding PLAN.md
   ```
   It clones the scaffold to a temp dir, asks you to confirm the
   component set (the *only* interactive step), copies `seed/` in,
   renders `CLAUDE.md` / `TODO.md` / `NEXT.md` / `TESTING.md` /
   `devctl.toml` from the plan, runs `git init` + `./setup.sh`, then
   discards the clone. `core` + `compose` + `tests` always install.
4. **Verify** the venv + CLI:
   ```
   ./setup.sh                       # idempotent; bootstrap already ran it
   .venv/bin/devctl --help          # lists subcommands incl. `test`
   ```
5. **Stand up the test harness first.** Bootstrap seeds a mandatory
   early epic *"Test harness + conventions"* in `TODO.md`. Do that epic
   before feature work: wire `[test]` in `tools/devctl.toml`, the
   tagging scheme, the long-lived test-DB reset, the `--ci` entrypoint.
   Read `TESTING.md` first.
6. **Work the epics.** `TODO.md` is authoritative. Per epic:
   ```
   tools/scripts/compose up                 # start the stack
   tools/scripts/run -- <build/cmd>          # captured one-shot
   tools/scripts/test --changed              # fast subset while iterating
   tools/scripts/test --full                 # before commit
   ```
   **Hard stop after each epic**: commit → update `TODO.md` + `NEXT.md`
   → wait. Resume a fresh session by reading `NEXT.md` first.

## Day-to-day commands (in a seeded project)

| Need | Command |
|------|---------|
| Start / stop stack | `tools/scripts/compose up` / `… down` |
| One-shot, captured | `tools/scripts/run -- <cmd>` |
| Tests | `tools/scripts/test [--unit\|--changed\|--ci\|--parallel]` |
| Interactive shell | `tools/scripts/shell` |
| Logs / status | `tools/scripts/logs` / `tools/scripts/ps` |
| Any ad-hoc, captured | `tools/scripts/capture -- <cmd>` |
| Wipe scratch | `tools/scripts/clean` |

Every captured command prints `LOG=<path>` first and `EXIT=<rc>` last
and writes a `manifest.json` + an `index.jsonl` line under
`tools/logs/` (gitignored). Read the `LOG=` path — never `| tail`.

## Add a component later

Skipped a component at bootstrap, or want to refresh one from the latest
scaffold:

```
/docker-compose-scaffolding-introduce <component>     # e.g. clean, diag
```

Resolves the component + deps from `seed/copy-manifest.yml`. Per file:
copies if absent; if it differs, shows a diff and asks
overwrite / skip / merge. No silent loss; only scaffold-owned paths are
touched.

## The two skills

Both live in this repo's `.claude/skills/` (installed once, above).

### `/docker-compose-scaffolding <plan-file>`

Bootstrap. Reads a **finalized** plan file, clones this repo to a temp
dir, confirms the component set with you (the only interactive step),
copies the seed into the current project, renders the `__TOKEN__`
templates (`CLAUDE.md` / `TODO.md` / `NEXT.md` / `TESTING.md`) +
`devctl.toml` from the plan, runs `git init` + `./setup.sh`, discards
the clone.

### `/docker-compose-scaffolding-introduce <component>`

Add or refresh one component later. Resolves the component + deps from
`seed/copy-manifest.yml`, and per file: copies if absent, or shows a
diff and asks overwrite / skip / merge if it differs. No silent loss.

## Repo layout

```
seed/                    what gets copied into target projects
  copy-manifest.yml      component → files + deps registry
  setup.sh pyproject.toml requirements.txt .gitignore
  CLAUDE.md TODO.md NEXT.md TESTING.md  rendered templates (__TOKEN__ slots)
  .claude/settings.json  generalized shim + read-only allowlist
  .claude/skills/dc-*    project-level skills
  tools/
    devctl.toml          per-project config
    docker-compose.yml   base stack (dev + test services)
    docker/Dockerfile.dev
    scripts/             shims → .venv/bin/devctl
    cli/devctl/          the CLI package
.claude/skills/          THIS repo's bootstrap + introduce skills
CLAUDE.md                guide for working IN this scaffold repo
```

## Components

| Component | Adds | Deps |
|-----------|------|------|
| `core`    | devctl (capture, show-tree, peek), venv bootstrap, config, tracking docs, allowlist | — |
| `compose` | base compose stack + dev image + compose/run/logs/ps + skills | core |
| `tests`   | TESTING.md + `devctl test` runner + dc-test skill | compose |
| `shell`   | interactive shell command + skill | compose |
| `clean`   | bounded scratch wipe + skill | core |
| `diag`    | ad-hoc captured-diagnostics skill | core |

`core` + `compose` + `tests` are always installed (testing is
mandatory); the rest are confirmed at bootstrap or added later via the
introduce skill.
