---
name: docker-compose-scaffolding-introduce
description: Add or update a single generic scaffolding component (e.g. shell, clean, diag, compose) in an already-scaffolded project, with per-file diff + overwrite/skip/merge prompts. Invoke as `/docker-compose-scaffolding-introduce <component>`. Use to pull in a scaffold component the project skipped at bootstrap, or to refresh one from the latest scaffold.
---

# docker-compose-scaffolding-introduce

Inject one new/updated generic component into the current
already-scaffolded project. No install-state file: installed components
are inferred from files present + the manifest.

`SCAFFOLD_URL = https://github.com/hnsk/claude-docker-compose-scaffolding`

## Inputs

- `$1` = component name (e.g. `shell`, `clean`, `diag`, `compose`).
  If missing/unknown, list manifest component names and stop.

## Procedure

1. **Clone** the scaffold shallowly: `git clone --depth 1 $SCAFFOLD_URL
   /tmp/dcs-intro-$$` (unique temp path; remember for cleanup).

2. **Read `seed/copy-manifest.yml`.** Resolve `<component>` + its
   `deps:` transitively into a file set. If a dep isn't installed yet,
   include it (it's required). Note: `core`, `compose`, and `tests` are
   always-installed at bootstrap; `tests` is only introduced here if a
   pre-`tests` project missed it.

3. **Per file**, compare scaffold copy vs the project's:
   - **Absent locally** → copy it in (note it).
   - **Present and identical** → skip silently.
   - **Present and differs** → show a unified `diff` (scaffold vs local)
     and **ask the user per file**: overwrite / skip / merge. On
     *merge*, apply the user's chosen combination explicitly; never lose
     local changes silently.

4. **Fix-ups:**
   - `chmod +x` any copied `tools/scripts/*` shims.
   - If a new `.claude/skills/dc-*` was added, update the skill table in
     the project's `CLAUDE.md`.
   - If `pyproject.toml` / the `devctl` package changed, re-run
     `./setup.sh`.
   - If new shims were added, add their `Bash(tools/scripts/<name>:*)`
     entries to the project's `.claude/settings.json` allowlist.
   - When `tests` is introduced: add
     `"Bash(tools/scripts/test:*)"` to the allowlist and the `dc-test`
     row to the `CLAUDE.md` skill table (parallels the existing
     skill/allowlist fix-ups). `TESTING.md` carries `__TOKEN__` slots —
     flag any left unrendered for the user to fill from the stack.

5. **Cleanup** the temp clone (`rm -rf /tmp/dcs-intro-$$`).

6. **Report** what was added/updated/skipped per file. End the turn.

## Notes

- Conflict policy is strictly per-file and user-driven — no bulk
  overwrite, no silent loss.
- This never touches the project's own source, only scaffold-owned
  paths from the manifest.
