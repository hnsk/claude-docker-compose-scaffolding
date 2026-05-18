# claude-docker-compose-scaffolding — repo guide (meta)

This repo is a **scaffold generator**, not an application. It has **no
runtime of its own**. Everything under `seed/` is copied verbatim into
*target* projects by the two skills in `.claude/skills/`.

## Mental model

- `seed/**` — exactly what lands in a seeded project. Edit here, never a
  generated copy elsewhere.
- `seed/` files carrying `__TOKEN__` slots (`CLAUDE.md`, `TODO.md`,
  `NEXT.md`, `TESTING.md`) are **templates** rendered at bootstrap.
  Preserve token syntax; don't hardcode project specifics.
- `seed/copy-manifest.yml` — the component → files + deps registry. **Any
  new `seed/` file must be added to a component**, or no project gets it.
- `.claude/skills/docker-compose-scaffolding/` (bootstrap) and
  `.../-introduce/` (add one component later) consume the manifest. The
  bootstrap skill also lists explicitly registered CLI subcommands and
  always-installed components — keep both in sync.
- `core` + `compose` + `tests` are always-installed; `tests` is
  mandatory (testing is not opt-in).

## Invariants — keep in sync on every change

A change to any one of these usually needs the others updated too:

1. New `seed/` file → `copy-manifest.yml` component.
2. New `devctl` subcommand → `commands/__init__.py` **and**
   `__main__.py` (registration there is explicit, not via `__all__`).
3. New shim → `seed/.claude/settings.json` allowlist + introduce-skill
   fix-up bullet.
4. New `dc-*` skill → `seed/CLAUDE.md` skill table + introduce fix-up.
5. New component / always-installed change → both skills + `README.md`
   (Components table + usage).
6. `README.md` is user-facing setup docs — update it for any
   workflow/component/skill change.

## Conventions

- **Latest-stable rule** in seed templates: never bake a
  training-memory image/runtime version; instruct verifying the latest
  stable at bootstrap.
- **Language-agnostic.** No Python/Node/C-specific notes in generic seed
  docs — stack specifics are filled at bootstrap from the plan.
- **Zero third-party runtime deps** in `devctl` (stdlib only;
  Python ≥ 3.11, `tomllib`).

## Verification

No runtime here. Verify changes in a throwaway seeded project: write a
minimal plan naming a stack, `/docker-compose-scaffolding <plan>` in
`/tmp/...`, run `./setup.sh`, exercise the changed path. Offline:
`python3 -m py_compile` + `PYTHONPATH=seed/tools/cli python3 -m devctl
--help`.

## Git

- Commit/branch/push only when the user asks.
- No `Co-Authored-By: Claude` line (user global rule).
