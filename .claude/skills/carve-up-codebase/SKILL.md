---
name: carve-up-codebase
description: |
  Split a monolithic code project (e.g. a single 1000-line file) into a clean,
  maintainable module structure — without breaking the repo. Use when the user
  asks to "refactor", "split into modules", "carve up the codebase",
  "modularize", "clean up the project structure", "separate concerns", or
  reports that "everything is in one file", "this file is too big", or "the
  project is hard to maintain". Applies to real code projects (TypeScript /
  JavaScript / Python / Go / Rust / Java / etc.) — NOT for docs, configs, or
  one-off scripts. Do NOT use for greenfield projects (just write modular code
  from the start), single-file bug fixes, or feature additions (use plain
  coding). Always preserves behaviour via a green-baseline test/build run
  before and after every change.
---

# Carve Up Codebase (Claude Code skill)

> The full procedure lives in `AGENTS.md` at this repo's root. This file is a
> pointer so Claude Code's skill loader picks it up from `.claude/skills/`.

## Procedure

Read and follow the full procedure in `AGENTS.md` (sibling repo root).

Quick recap — five phases, all mandatory:

1. **Baseline** — green test/build state, worktree isolated
2. **Plan** — read file, propose module tree, get user approval
3. **Apply incrementally** — one cluster per commit, every commit green
4. **Verify** — full suite + orphan-import + circular-dep scan
5. **Hand off** — summary + merge instructions

## Safety rules (do not skip)

- Green baseline BEFORE any edits. Refusing to start on red is non-negotiable.
- One cluster per commit. Each commit green at HEAD. No "fix in next commit".
- Worktree isolation. User's working tree is read-only during the refactor.
- Refactor without tests = guessing. Stop and ask for a smoke test.
- Behaviour-preserving only. No feature adds, no optimisations, no test edits.

## Reference material (shared at repo root)

- `../../references/heuristics.md` — per-language module-splitting rules
- `../../references/verification.md` — per-stack test/build command discovery

For the complete procedure, read `../../../AGENTS.md`.