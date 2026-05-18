# __PROJECT_NAME__ — testing conventions

__PROJECT_SUMMARY__

Authoritative testing policy for this project. Seeded from the
`claude-docker-compose-scaffolding` scaffold and adopted to this stack at
bootstrap. This repo owns and may mutate it. The runner is
`tools/scripts/test` (skill: `dc-test`); policy lives here.

## Mandatory

- **All new or changed code ships tests.** Unit tests for logic;
  **integration tests for anything crossing a service / DB / network
  boundary.** A change without tests is incomplete, not "to be tested
  later".
- A test added for a bug reproduces that bug (fails before the fix).
- Tests run **only** inside the compose `test` profile — never on the
  host. See the Golden rules in `CLAUDE.md`.

## Tagging taxonomy (from day one)

Every test carries **two independent tags** so a small change runs a
small subset instead of a 30-minute full suite:

1. **Speed tier** — exactly one of:
   - `unit` — no I/O, no DB, no network. The default fast set.
   - `integration` — touches DB / a service / the filesystem.
   - `e2e` — exercises the full stack.
2. **Area / module** — one per-module tag (e.g. the package or feature
   name) so a scoped change runs *just that area* + the whole `unit`
   tier.

Concrete tag syntax for this stack:

```
__TEST_TAG_SYNTAX__
```

Tagging is **not optional and not retrofitted** — an untagged test is a
broken test. New modules add their area tag on the first test.

## Selective execution policy

| Trigger                         | Runs                                    |
|---------------------------------|-----------------------------------------|
| Scoped code change (dev loop)   | that area's tag **+** the `unit` tier   |
| Pre-commit                      | full suite (`--full`)                   |
| Merge / main                    | full suite                              |
| Nightly                         | full suite incl. `e2e`                  |

- Dev loop: `tools/scripts/test --changed` (or `--unit` for the fast
  tier only). `--changed` substitutes the changed-file list for a
  `{files}` token in `[test].changed`.
- **Native impact tool**, when this stack has one (preferred over tags
  for the dev loop, tags remain the portable fallback):

  ```
  __TEST_IMPACT_TOOL__
  ```

- Pre-commit / merge / nightly always run the **full** suite — selective
  runs are a dev-loop speed optimization, never the gate.

## Parallel-safety rules

Tests run in parallel by default; an order- or state-dependent test is a
bug.

- **No shared mutable state across tests**: no shared DB rows, files,
  fixed ports, or singletons.
- Each parallel **worker** gets an **isolated schema / database** (or
  strictly disjoint data) — never a shared one.
- Per-worker scratch goes under repo-root **`./tmp/<worker>/`**
  (gitignored), never a hardcoded `/tmp` path.
- No test depends on another test or on execution order. No "test 2
  assumes test 1 ran".
- Bind ephemeral ports (`:0`) or per-worker fixed ports; never a single
  global port.

## Database / service strategy

The test DB / cache is a **long-lived compose service under the `test`
profile** — started **once** and **never restarted per run**. Per-run
container recreation is slow and is explicitly *not* how state is reset.

- State is reset **between tests** by **`TRUNCATE` or
  transaction-rollback**, not by recreating the container.
- Parallel isolation: a **schema or database per worker** inside that
  one long-lived service.
- Schema/migrations are applied once when the service comes up; tests
  assume it is already there.

Concrete reset mechanism for this stack:

```
__TEST_DB_RESET__
```

The long-lived test-DB service (if this project needs a DB) is defined
in `tools/docker-compose.yml` under the `test` profile with a named
volume and **no per-run restart**.

## CI strategy

CI is **provider-agnostic**: it calls exactly one captured entrypoint —

```
tools/scripts/test --ci
```

— and nothing else. No provider-specific test logic; switching CI
providers never changes how tests run.

- Push / PR → `[test].ci` should run the **`--changed` fast subset**
  (quick signal).
- Merge / main → **full** suite (the gate).
- A CI provider workflow file is added to this repo **only if** the
  project's plan named a provider; otherwise the entrypoint stays
  provider-neutral and any provider can call it.

## Runner reference

`tools/scripts/test` (skill `dc-test`) wraps
`docker compose run --rm <[test].service> <cmd>` in capture. Modes map to
`[test]` keys in `tools/devctl.toml`:

| Flag        | Config key       | Use                          |
|-------------|------------------|------------------------------|
| `--full`    | `[test].command` | whole suite (default)        |
| `--unit`    | `[test].unit`    | fast tier                    |
| `--changed` | `[test].changed` | impacted (`{files}` token)   |
| `--ci`      | `[test].ci`      | CI entrypoint                |
| `--parallel`| `[test].parallel`| append parallel-run args     |

`--unit/--changed/--ci` fall back to `[test].command` when their key is
empty; all empty → the runner exits 2 (nothing wired yet — that is the
**"Test harness + conventions"** epic in `TODO.md`).
