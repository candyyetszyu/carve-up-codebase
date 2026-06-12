# Carve Up Codebase

Split a monolithic code project (e.g. a single 1000-line file) into a clean,
maintainable module structure — **without breaking the repo**.

A portable skill repo that works across the major AI coding agents: Cursor,
Codex, Claude Code, Aider, Copilot, Gemini CLI, Windsurf, Zed, JetBrains
Junie, Mavis, and any other tool that reads the open
[`AGENTS.md`](https://agents.md) standard.

> **TL;DR** — when an AI agent sees `AGENTS.md` (or its tool-specific skill
> file) in your project, it knows how to carve a giant single-file project
> into modules safely: green baseline → user-approved plan → one cluster per
> commit → every commit green. Copy `AGENTS.md` into any project root and
> ask the agent to "carve up the codebase".

---

## Introduction

`carve-up-codebase` is a guardrailed workflow for the moment when a project's
source has outgrown a single file — 1000+ line `app.ts`, a 50KB Python module,
a giant `main.go` that does seven things. The instinct is to refactor; the
risk is breaking the build, losing tests, or moving code into worse boundaries
than you started with.

This skill fixes that by enforcing a five-phase procedure:

1. **Baseline** — capture the green state of test/build/lint before editing
   anything. Refactor without a green baseline is guessing.
2. **Plan** — read the file, propose a module tree, **get explicit user
   approval** before touching code.
3. **Apply** — move clusters one at a time. Each move is its own commit, and
   every commit must leave the project green. No "fix in next commit" chains.
4. **Verify** — re-run the full suite, scan for orphan imports and circular
   deps, surface any leftover tech debt.
5. **Hand off** — work happens on an isolated worktree / branch, so the user's
   main tree stays clean until they merge.

The same content is published in every format major coding agents understand
natively — `AGENTS.md` for the open standard, plus `.claude/skills/`,
`.codex/skills/`, and `.cursor/rules/` wrappers — so you can drop it into a
project and have any agent pick it up without extra wiring.

**Use it when:** a file is oversized, mixed-concern, or "hard to maintain" and
you want it restructured safely. **Don't use it for:** greenfield projects,
single bug fixes, or feature work — those have their own procedures.

---

## What it does

Given a real code project (TypeScript / JavaScript / Python / Go / Rust /
Java / etc.) where one or more files have grown too large or too tangled, the
skill will:

1. **Baseline** — capture the green state of the test/build/lint suite, so
   "didn't break" is provable, not promised.
2. **Plan** — read the file end-to-end, propose a target module tree, and get
   your explicit approval before touching anything.
3. **Apply** — move clusters into new modules one at a time. Each move is a
   commit, and each commit leaves the project green. No "fix in next commit"
   chains.
4. **Verify** — re-run the full suite, scan for orphan imports and circular
   deps, hand off a clean refactor branch.
5. **Don't touch your working tree** — all work happens on a separate worktree
   / branch (`refactor/<name>`) so you can review the diff before merging.

The deliverable is a refactor branch where **every previously-passing check
still passes** and behaviour is identical.

---

## When to trigger it

The skill auto-loads when you say things like:

- "Refactor this project"
- "Split this file into modules"
- "Carve up the codebase"
- "Modularize the codebase"
- "This file is too big, clean it up"
- "Separate concerns — everything is in one file"
- "Make this maintainable"

It is **code-specific** — it won't try to refactor docs, configs, or one-off
scripts. If you ask it to refactor something with no tests and no build, it'll
stop and ask you to add a smoke test first, because refactor without
verification is guessing.

---

## Quick start — pick your agent

This repo is structured so the **same skill content is reachable from any
major coding agent** without modification. Pick the one you use:

### Cursor

Drop this repo's `.cursor/rules/carve-up-codebase.mdc` into your own project
at `<your-project>/.cursor/rules/carve-up-codebase.mdc`. Cursor auto-loads
it.

Or just copy the `AGENTS.md` content into your project root — Cursor also
reads `AGENTS.md` natively.

### Codex

Drop `.codex/skills/carve-up-codebase/SKILL.md` into
`~/.codex/skills/carve-up-codebase/SKILL.md` (global) or
`<your-project>/.codex/skills/carve-up-codebase/SKILL.md` (project).

Or just copy `AGENTS.md` into your project root — Codex also reads
`AGENTS.md` natively.

### Claude Code

Drop `.claude/skills/carve-up-codebase/SKILL.md` into
`~/.claude/skills/carve-up-codebase/SKILL.md` (global) or
`<your-project>/.claude/skills/carve-up-codebase/SKILL.md` (project).

Or just copy `AGENTS.md` into your project root and add `@AGENTS.md` to your
`CLAUDE.md` — Claude Code will pull it in.

### Mavis (mavis)

Symlink or copy `.claude/skills/carve-up-codebase/SKILL.md` into
`~/.mavis/skills/carve-up-codebase/SKILL.md`. Mavis reads Claude Code's skill
format natively.

Or use `AGENTS.md` directly — Mavis also understands the open standard.

### Aider / Copilot / Gemini CLI / Windsurf / Zed / Junie

Just copy `AGENTS.md` into your project root. All these tools read it
natively. Done.

---

## File layout

```
carve-up-codebase/
├── AGENTS.md                            # open-standard entry — works in
│                                          Cursor, Codex, Aider, Copilot,
│                                          Gemini CLI, Windsurf, Zed, Junie,
│                                          Mavis, Claude Code (via @import)
├── README.md                            # this file (human-facing)
├── references/
│   ├── heuristics.md                    # per-language module-splitting rules
│   │                                      (TS/JS, Python, Go, Rust, Java)
│   └── verification.md                  # per-stack test/build command
│                                          discovery
├── .claude/
│   └── skills/
│       └── carve-up-codebase/
│           └── SKILL.md                 # Claude Code's native skill format
├── .codex/
│   └── skills/
│       └── carve-up-codebase/
│           └── SKILL.md                 # Codex's native skill format
└── .cursor/
    └── rules/
        └── carve-up-codebase.mdc        # Cursor's native .mdc rule format
```

`AGENTS.md` is the source of truth. The tool-specific files (`SKILL.md` in
.claude/ and .codex/, `carve-up-codebase.mdc` in .cursor/) point to it — so
any update to `AGENTS.md` flows to every agent.

---

## Safety guarantees

The skill is designed around three invariants:

1. **Green baseline before edits.** No refactor starts until the test/build
   suite is green. If it's red, the skill stops and tells you.
2. **Per-cluster commits.** Each moved chunk is one commit. Every commit is
   green at HEAD. If a commit goes red, the skill reverts it and debugs —
   never "fix in the next commit".
3. **Worktree isolation.** The skill works in a separate git worktree, so
   your main working tree stays untouched until you merge the refactor
   branch.

These are not aspirational — they're enforced in the procedure the skill
follows. If a step is skipped, the skill is misbehaving.

---

## What it does NOT do

- **Add features.** A refactor is definitionally behaviour-preserving. If
  you want a new feature, that's a separate task.
- **Optimise.** "Move the code verbatim, optimise later in another PR."
- **Touch tests as part of the move.** If the move breaks a test, the move
  is wrong — fix the move, not the test.
- **Refactor without tests.** If your project has no tests, the skill stops
  and asks you to add a smoke test that exercises the public surface first.
- **Refactor a single 50-line script.** Cohesive small files are fine — don't
  fragment them just because the skill exists.

---

## Install — pick one

**Option A — project-scoped (recommended for teams)**

Copy this repo's `.cursor/`, `.claude/`, and/or `.codex/` folders, plus
`AGENTS.md`, into your project. Commit to git so the whole team shares it.

**Option B — globally installed (just for you)**

Symlink the tool-specific skill files into your home directory's tool
config. For example, for Claude Code:

```bash
ln -s "$(pwd)/.claude/skills/carve-up-codebase" ~/.claude/skills/carve-up-codebase
```

For Codex:

```bash
ln -s "$(pwd)/.codex/skills/carve-up-codebase" ~/.codex/skills/carve-up-codebase
```

For Cursor (project rules, not global):

```bash
# In each project where you want this rule:
mkdir -p .cursor/rules
cp "$(pwd)/.cursor/rules/carve-up-codebase.mdc" .cursor/rules/carve-up-codebase.mdc
```

For Aider, Copilot, Gemini CLI, Windsurf, Zed, Junie:

```bash
# Just copy AGENTS.md into your project root:
cp "$(pwd)/AGENTS.md" /path/to/your/project/AGENTS.md
```

---

## Updating the skill

Edit `AGENTS.md` at the repo root. The tool-specific files all point to it,
so your change flows to every agent. If you add a major section, also update
the tool-specific wrapper files so their descriptions still match.

---

## See also

- [`AGENTS.md` open standard](https://agents.md) — the cross-agent format
  this skill is built on
- [Claude Code skills docs](https://docs.anthropic.com/en/docs/claude-code/skills)
- [Cursor rules docs](https://docs.cursor.com/welcome)
- [OpenAI Codex docs](https://developers.openai.com/codex)

---

## License

MIT — do whatever you want with it. If you improve it, PRs welcome.

---

## Suggested initial commit message

A ready-to-paste commit message for the initial import:

```
feat: initial carve-up-codebase skill

Portable skill repo that splits monolithic code projects into a maintainable
module structure without breaking the repo. Works across the major AI coding
agents via the open AGENTS.md standard.

Content:
- AGENTS.md              # source of truth (Cursor, Codex, Aider, Copilot,
                           Gemini CLI, Windsurf, Zed, Junie, Mavis, etc.)
- README.md              # human-facing install + usage
- references/            # per-language heuristics + per-stack verification
- .claude/skills/        # Claude Code native skill format
- .codex/skills/         # Codex native skill format
- .cursor/rules/         # Cursor native .mdc rule format

Five-phase procedure (baseline → plan → apply → verify → hand off) enforces
green tests at every commit, worktree isolation, and explicit user approval
before any edits.
```

Short version (if your repo follows a strict subject-line-only convention):

```
feat: add carve-up-codebase skill (multi-agent, AGENTS.md standard)
```

Commit it with:

```bash
cd /Users/judyyip/coding-agent-skills/carve-up-codebase
git init
git add -A
git commit -F- <<'MSG'
feat: initial carve-up-codebase skill

Portable skill repo that splits monolithic code projects into a maintainable
module structure without breaking the repo. Works across the major AI coding
agents via the open AGENTS.md standard.

Five-phase procedure (baseline → plan → apply → verify → hand off) enforces
green tests at every commit, worktree isolation, and explicit user approval
before any edits.
MSG
```