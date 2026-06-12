# Verification — Finding the Right Test / Build Command

The verification suite is the contract that proves "didn't break". Before any
refactor, you must know how to run it, capture a green baseline, and re-run
it after every cluster move.

## Step 1: Detect the project's verification surface

Look for these files in order of priority:

| Signal | Meaning |
|---|---|
| `package.json` (`scripts.test`, `scripts.build`) | npm/pnpm/yarn project — test/build commands are right there |
| `pyproject.toml` (`[tool.pytest]`, `[project.optional-dependencies]`) | Python with pytest |
| `setup.py` / `setup.cfg` | Older Python packaging |
| `Cargo.toml` | Rust — `cargo test` + `cargo build` |
| `go.mod` | Go — `go test ./...` + `go build ./...` |
| `pom.xml` / `build.gradle` | Java/Maven/Gradle |
| `Makefile` / `justfile` / `taskfile.yml` | Generic — look for `test:`, `build:`, `check:` targets |
| `Gemfile` | Ruby — `bundle exec rspec` or `bundle exec rake test` |
| `.github/workflows/*.yml` | CI definitions often show the canonical commands — grep for `run:` lines |
| `package.json` (`scripts.lint`, `scripts.typecheck`) | TS/JS lint+typecheck, run alongside test |

If none of these exist, **stop**. Tell the user to add at least a smoke test
before any refactor.

## Step 2: Identify ALL checks, not just tests

A "green baseline" usually requires running multiple commands:

- **Unit tests** (the obvious one)
- **Build** (catches import errors, missing files)
- **Type check** (TS: `tsc --noEmit`, Python: `mypy`, Rust: `cargo check`)
- **Lint** (catches unused imports the move may introduce)

Find them all. Run them all on the baseline. Run them all after each move.

## Step 3: Build a single verification script

Don't re-type four commands after every commit. Create a `verify.sh` (or
`verify.ps1` on Windows) at the project root that runs them all:

```bash
#!/usr/bin/env bash
set -e
echo "→ typecheck"
npm run typecheck
echo "→ lint"
npm run lint
echo "→ test"
npm test
echo "→ build"
npm run build
echo "✓ all green"
```

Commit this script as part of the refactor branch. Now the per-cluster check
is one command. Delete it after the refactor merges if you don't want to keep
it.

## Stack-specific recipes

### npm / pnpm / yarn (Node / TS)

```bash
# Baseline
npm run typecheck && npm run lint && npm test && npm run build

# Per-cluster verify (after each commit)
bash verify.sh
```

Watch out for:
- `npm test` may be silent if no test framework is configured (`echo "Error:
  no test specified"`). Check that it actually runs something.
- `npm run build` may be slow (10s+ for TS projects). Tolerate it but don't
  skip it — builds catch real issues.

### Python (pytest)

```bash
# Baseline
pytest -q
# or with mypy
mypy src && pytest -q

# Per-cluster
bash verify.sh
```

Watch out for:
- `pytest` discovers by config. Run from project root.
- For `src/` layout, pytest may need `pip install -e .` first.

### Go

```bash
# Baseline
go vet ./... && go test ./... && go build ./...

# Per-cluster — Go is fast, just run all three every time.
```

Go's `go build ./...` is the strongest import-graph check. If imports are
broken, this fails.

### Rust

```bash
# Baseline
cargo check && cargo test && cargo clippy -- -D warnings

# Per-cluster
```

`cargo check` is much faster than `cargo build` and catches the same import
issues. Use it during iteration; `cargo build` only at the end.

### Java / Maven

```bash
# Baseline
mvn -q test
# or with checkstyle / spotbugs
mvn -q verify

# Per-cluster
```

### Make-based projects

```bash
make test
make build
# or whatever targets the Makefile defines
```

## When the suite is slow (>5 min)

Some projects have heavy test suites. Strategies:

1. **Subset the tests for iteration.** Run only the tests touching the moved
   cluster during per-cluster verification. Run the full suite at the end.
2. **Use cache.** Many tools (`cargo`, `tsc --build`, gradle) cache build
   artifacts; second runs are fast.
3. **Run tests in parallel.** `pytest -n auto`, `cargo test -- --test-threads`,
   `go test -parallel`.
4. **If even the subset is >2 min**, run them async and check results when
   they finish. Don't sit idle.

## Baseline capture

Save the baseline output to a file:

```bash
mkdir -p .refactor
( npm run typecheck && npm run lint && npm test && npm run build ) \
  > .refactor/baseline.log 2>&1 \
  && echo "✓ baseline green" \
  || ( echo "✗ baseline RED — aborting refactor" && exit 1 )
```

If baseline is red, **stop the refactor**. Tell the user what's failing.
Refactor on a red baseline means you can't tell whether new failures are
yours or pre-existing.

## When something fails AFTER a cluster move

1. `git diff HEAD~1 --stat` → see exactly what changed
2. `git diff HEAD~1` → see the actual diff
3. Common causes:
   - Forgot to update a relative import path
   - New file is in a directory without an `__init__.py` / `index.ts`
   - Circular import introduced
   - Re-export missed a symbol
4. Fix and recommit. If you can't figure it out in 2 attempts, `git reset
   --hard HEAD~1` and try a different cut for that cluster.

## Final verification

Before declaring done:

```bash
git diff main --stat    # confirm only intended files
git diff main           # eyeball the diff
bash verify.sh         # full suite, must be green
rg "from ['\"].*oldpath"   # orphan imports must be empty (or only shim)
```

If any of these surfaces an issue, fix before reporting done.