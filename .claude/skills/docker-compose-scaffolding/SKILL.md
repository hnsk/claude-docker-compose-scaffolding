---
name: docker-compose-scaffolding
description: Seed the current project with the generic Docker-Compose scaffolding (devctl CLI, captured logs, dc-* skills, TODO/NEXT epic workflow) from a finalized plan file. Invoke as `/docker-compose-scaffolding <plan-file>`. Use when starting a new project that should adopt the all-in-Docker-Compose, AI-friendly-logging convention.
---

# docker-compose-scaffolding (bootstrap)

Consume a **finalized** plan file and seed the current directory (the
target project) with the scaffold. You are a *consumer* of the plan, not
its author: do **not** re-litigate requirements or re-plan. The only
interactive step is confirming the component set.

`SCAFFOLD_URL = https://github.com/hnsk/claude-docker-compose-scaffolding`

## Inputs

- `$1` = path to the plan file (the finalized brief: what to build,
  languages, services needed). If missing, ask for it and stop.

## Procedure

1. **Read the plan file.** Treat it as the finalized brief. Extract:
   project name, one-paragraph summary, languages/runtimes, services
   needed (db? cache? test runner?), and the work breakdown.

2. **Clone the scaffold** shallowly to a temp dir:
   ```
   tools/scripts # not yet present — use a raw command:
   git clone --depth 1 $SCAFFOLD_URL /tmp/dcs-seed-$$
   ```
   (Use a unique temp path; remember it for cleanup.)

3. **Read `/tmp/dcs-seed-$$/seed/copy-manifest.yml`.** Always select
   `core` + `compose` + `tests` (testing is mandatory, never opt-in).
   Infer optional components from the plan:
   - `shell` — almost always useful (interactive debugging).
   - `clean` — if the project produces scratch/build artifacts.
   - `diag` — almost always useful (ad-hoc captured diagnostics).
   Then **AskUserQuestion** to confirm the final component set (this is
   the only interactive step). Resolve `deps:` transitively.

4. **Copy** the union of selected components' files from
   `/tmp/dcs-seed-$$/seed/<path>` into the target project (cwd),
   preserving paths. A manifest path ending in `/` = copy the whole
   subtree. Keep script executability (`chmod +x tools/scripts/*`,
   `setup.sh`).

5. **Render templates** from the plan (replace every `__TOKEN__`):
   - `CLAUDE.md` — `__PROJECT_NAME__`, `__PROJECT_SUMMARY__`.
   - `TESTING.md` — render `__PROJECT_NAME__`, `__PROJECT_SUMMARY__`,
     and the stack slots from the plan's stack:
     - `__TEST_TAG_SYNTAX__` — concrete tag syntax (pytest markers /
       jest projects / go build tags / …).
     - `__TEST_IMPACT_TOOL__` — the stack's native test-impact tool if
       it has one (e.g. `pytest --testmon`, `jest --findRelatedTests`,
       Go package graph); else state "none — rely on the area tags".
     - `__TEST_DB_RESET__` — concrete reset mechanism (TRUNCATE all
       tables / wrap each test in a rolled-back transaction / per-worker
       schema), matching the plan's DB.
     Leave **no** stray `__TOKEN__`.
   - `TODO.md` — turn the plan's work breakdown into epics. Each epic =
     a small, independently-committable chunk with a checklist. Replace
     the `__EPIC_*__` tokens; add as many epics as the plan implies.
     **Seed a mandatory early epic** "Test harness + conventions" (wire
     the concrete `[test]` runner, the tagging scheme, parallel-safe
     long-lived test-DB reset, the `--ci` entrypoint) so every project
     stands testing up *before* feature work.
   - `NEXT.md` — point at Epic 1 / its first item.
   - `tools/devctl.toml` — set `[compose].default_service`,
     `[clean].targets`, and the full `[test]` block from the plan's
     concrete runner: `service`, `command` (--full), `unit`, `changed`
     (use a `{files}` token unless the tool self-tracks), `ci`,
     `parallel`. Leave defaults only if the plan truly says nothing.
   - `tools/docker-compose.yml` — if the plan needs a DB, uncomment /
     fill the **long-lived `test-db`** service under the `test` profile
     (named volume, no per-run restart; pin the latest stable image tag,
     verified from Docker Hub).
   - `pyproject.toml`, `.gitignore` — only adjust if the plan clearly
     requires it (these are devctl's own, not the project's; usually
     leave as-is).
   - `tools/docker/Dockerfile.dev` — if the plan names a language, add
     the runtime under the marker, **pinned to the latest stable**
     version. Verify the current latest from the official source /
     Docker Hub (run the check under `git`/`curl` and note it); do not
     use a training-memory version.

6. **Init + bootstrap.** `git init` if not already a repo. Run
   `./setup.sh` (builds `.venv`, installs `devctl`). Confirm
   `.venv/bin/devctl --help` works.

7. **Cleanup.** Remove the temp clone (`rm -rf /tmp/dcs-seed-$$`).

8. **Report.** Print: components installed, the generated epic list, and
   the hard-stop reminder:

   > TODO.md is authoritative. Work one epic at a time. After each epic:
   > commit → update TODO.md + NEXT.md → **stop and wait for the user**.

   Then end the turn (do not start Epic 1).

## Notes

- The scaffold is a seed, not a dependency — never re-clone it for this
  project after bootstrap. To add a component later, the project uses
  `/docker-compose-scaffolding-introduce <component>`.
- **CI stays provider-agnostic.** The captured entrypoint is always
  `tools/scripts/test --ci`. Add a CI-provider workflow file (e.g.
  `.github/workflows/`) that calls *only* that **only if the plan names
  a provider**; otherwise add no workflow file and leave CI neutral so
  any provider can call the entrypoint.
- Don't copy `seed/`-external files (the bootstrap/introduce skills,
  the scaffold's own README/PLAN). Only `seed/<...>` paths.
