---
name: carve-up-codebase
description: |
  Split a monolithic code project (e.g. a single 1000-line file) into a clean,
  maintainable module structure — without breaking the repo. Use when the user
  asks to "refactor", "split into modules", "carve up the codebase", "modularize",
  "clean up the project structure", "separate concerns", or reports that "everything
  is in one file", "this file is too big", or "the project is hard to maintain".
  Applies to real code projects (TypeScript / JavaScript / Python / Go / Rust /
  Java / etc.) — NOT for docs, configs, or one-off scripts. Do NOT use for
  greenfield projects (just write modular code from the start), single-file bug
  fixes, or feature additions (use plain coding). Always preserves behaviour via
  a green-baseline test/build run before and after every change.
---

# Carve Up Codebase

Split a working monolithic code project into a clean module layout **without
breaking it**. The contract: after the refactor, every previously-passing check
(test, build, lint, type-check) still passes, and behaviour is identical.

## When this applies

- Repo has one or more oversized source files (typically 500+ lines, single
  responsibility violated)
- The user wants maintainability, not new features
- Tests / build / type-check already pass before starting (the "green baseline")
- Project is real code with imports between files (not a script, not docs)

## When NOT to use

- Greenfield project → just write modular code from the start
- Single bug fix → fix it directly
- Adding a new feature → feature work, not modularization
- Repo has no tests and no build (nothing to verify "didn't break") → ask the
  user to add at least a smoke test before starting

## Inputs to collect (before doing anything)

1. **Project root path** — where the source code lives
2. **The target file(s) to split** — usually 1–3 oversized files
3. **Language / framework** — affects module conventions (TS uses
   `import`+`index.ts`, Python uses packages with `__init__.py`, Go uses
   packages, etc.)
4. **Test command and build command** — non-negotiable. No tests = no refactor
5. **Risk tolerance** — confirm: worktree / branch, or straight to main?
   **Default: worktree.**

If any of these are missing and you can't infer them from the repo, ask once
and concisely. Do not start modularizing without (1) (2) (3) (4).

## Procedure

### Phase 1 — Baseline (mandatory, no edits yet)

1. Confirm the project is a git repo. If not, stop and ask the user to
   `git init` first — we need git history as the safety net.
2. Create a worktree:
   `git worktree add ../<project>-carve -b refactor/<short-name>`
   (or a new branch if the user prefers no worktree). **All work happens here,
   never on the user's working tree.**
3. Run the full verification suite (test + build + lint + type-check) and
   record the result. This is the **green baseline**. If anything fails, stop
   and report — cannot start from a red baseline.
4. Snapshot the dependency graph of the target file: every file that imports
   it, every external module it depends on. Use `grep` / `rg` for this. The
   refactor must keep every import working.

### Phase 2 — Module plan

1. Read the target file end-to-end. Identify natural seams:
   - Top-level functions / classes that share a theme (e.g. all "user
     authentication" stuff → `auth.ts`)
   - Internal state that only one cluster touches (move with that cluster)
   - Pure helpers (utilities) → `utils.ts` or `helpers/`
   - Types / interfaces used by multiple clusters → `types.ts`
2. Propose a target module tree. Show the user before editing anything.
   Example for a 1000-line `app.js`:
   ```
   src/
     index.ts              # public entry, re-exports
     auth/
       index.ts
       login.ts
       session.ts
     users/
       index.ts
       repository.ts
       validators.ts
     utils/
       index.ts
     types.ts
   ```
3. **Get explicit user approval on the plan.** The plan must list:
   - What moves where (file → file mapping)
   - What new public API surface looks like
   - Backward-compat strategy: keep a thin re-export shim at the old location,
     or rename all call sites? (Default: keep shim, deprecate later.)
4. If the user wants a different cut, iterate the plan until they sign off.

### Phase 3 — Apply incrementally (never the whole file at once)

Apply the move in the smallest possible commits. For each cluster:

1. **Create the new module file(s)** with the extracted code. Include only the
   cluster's own imports (don't bring in unused deps).
2. **Update the old file** to import from the new location (or re-export for
   back-compat). Don't delete from the old file yet.
3. **Run the verification suite.** If green, commit. If red, revert this
   cluster's changes (`git checkout -- .` in the worktree) and debug.
4. After all clusters moved and verified green, delete the now-empty code
   blocks from the old file. Re-run verification. Commit.

The key rule: **each commit must leave the project green.** No "fix in next
commit" — if the suite is red after a commit, that commit is wrong, revert it.

### Phase 4 — Verify and clean up

1. Re-run the full verification suite one more time on the final state.
2. `git diff --stat` from the baseline branch → confirm only intended files
   changed.
3. Look for any orphaned imports (`rg "from ['\"].*oldpath"` should be empty or
   only contain the back-compat shim).
4. Look for any dead code the move exposed (functions no longer called from
   outside their original location — usually a sign they should move with
   their caller).
5. Report back to the user:
   - What moved where (one-line per file)
   - Test/build result (must say "green" — if not, do not declare done)
   - Any leftover tech debt spotted (orphans, circular deps, oversized
     helpers)
   - Whether to keep or remove the back-compat shim (recommend: keep for one
     release, then remove)

### Phase 5 — Hand off

Tell the user how to merge the worktree back: which branch to merge into,
whether to squash or keep the per-cluster commits (recommend keep — each is
self-contained green).

## Output contract

- A refactored worktree / branch where every previously-passing check still
  passes
- One commit per logical cluster move, each green at HEAD
- A short written summary: file-by-file what moved, what's the new public API,
  what (if anything) is left as back-compat shim
- No code in the user's working tree was modified — all changes live on the
  refactor branch

## Failure handling

- **Verification fails after a cluster move** → revert that cluster
  (`git checkout -- <files>` or `git reset --hard HEAD~1`), debug the import
  / circular dep. Do not move on until green.
- **User has no tests** → stop. Ask them to add a smoke test that exercises
  the target file's public API before continuing.
- **Circular import shows up after move** → split the offending module
  further (the cycle usually means two clusters were forced into one file
  originally). Re-plan only that cluster.
- **File is genuinely cohesive (not actually too big)** → tell the user. Don't
  refactor for the sake of refactoring. A 1000-line file with one tight
  concern is better than five 200-line files with awkward boundaries.
- **Plan approved but you realise mid-way it needs to change** → stop, surface
  the new finding, re-confirm. Don't silently re-plan after Phase 2.
- **Worktree creation fails (uncommitted changes, etc.)** → tell user to
  `git stash` or commit first, then retry.

## Examples

**Example 1 — User reports a 1000-line `app.ts`**

User: "My Next.js app has everything in `app.ts` (1200 lines). Can you split it
into modules? Don't break the build."

→ Detect: Next.js (TS), `pnpm test` + `pnpm build` for verification, create
worktree, run baseline (assume green), propose split into `auth/`, `data/`,
`ui/`, `utils/`, get approval, apply cluster-by-cluster, each commit green,
report summary.

**Example 2 — User asks "carve up my Python script"**

If it's actually a single 50-line script with no tests, this skill is wrong —
push back. The user probably wants a feature add or a bug fix. If it's a
real package with multiple oversized modules, proceed normally.

## Reference material

- `references/heuristics.md` — module-splitting heuristics by language
  (TS/JS, Python, Go, Rust, Java) — when to extract, what to put in
  `index.ts`, circular-import patterns to watch for.
- `references/verification.md` — how to discover the right test / build
  command for common stacks (npm/pnpm/yarn, pytest, go test, cargo, mvn),
  and what to do when the suite is slow.