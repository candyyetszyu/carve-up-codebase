# Module-Splitting Heuristics by Language

Practical guidance for finding module seams. These are heuristics, not rules —
the goal is maintainability, not purity. Sometimes "leave it alone" is the
right answer.

## Universal signals that a file should split

A file is a refactor candidate when ANY of these are true:

- **Length**: >500 lines is a strong signal; >1000 is almost always.
- **Mixed abstraction levels**: top-level types + business logic + HTTP
  handlers + DB queries + utility functions all in one file.
- **Multiple distinct import groups**: if `import { x } from 'a'` and
  `import { y } from 'b'` and `import { z } from 'c'` are all unrelated, those
  groups probably belong in different modules.
- **Comments like "// auth stuff" or "// TODO: separate"** sprinkled through
  the file. The author already saw the seam.
- **Hard to name**: if you can't describe the file in one short phrase
  ("this is the user service") without "and also", it's doing too much.

## TypeScript / JavaScript

### Default target layout

```
src/
  index.ts            # public entry — re-exports the public API
  <feature>/
    index.ts          # barrel for the feature
    <sub-concern>.ts  # actual implementations
  types.ts            # shared types/interfaces
  utils/              # pure helpers, no side effects
    index.ts
```

### Barrel pattern (`index.ts` re-exports)

When splitting, each new subdirectory gets an `index.ts` that re-exports the
public symbols. Other modules import from the directory, not the file:

```ts
// auth/index.ts
export { login } from './login';
export { session } from './session';
export type { AuthConfig } from './types';
```

Callers then do `import { login } from './auth'` not `import { login } from
'./auth/login'` — the file path is an implementation detail.

### Circular import patterns to watch for

- Two modules that import each other's types only → extract the types to a
  shared `types.ts`.
- A module that imports a function and exports a wrapper → invert: have the
  caller wrap, not the module.
- `index.ts` files importing from siblings that import from the same
  `index.ts` → break the cycle by moving one symbol out of the barrel.

### What to put in `utils/`

Only **pure** functions (no I/O, no module-level state, no side effects).
Anything that talks to a database, network, filesystem, or `Date.now()` is
NOT a util.

## Python

### Default target layout

```
src/<package>/
  __init__.py         # public API — re-exports from submodules
  <feature>.py        # or subpackages for bigger concerns
  types.py
  utils.py
```

For larger projects:

```
src/<package>/
  __init__.py
  auth/
    __init__.py
    login.py
    session.py
  users/
    __init__.py
    repository.py
  utils.py
```

### `__init__.py` as public surface

`__init__.py` should re-export only the public API:

```python
from .auth import login, session
from .users import User, UserRepository

__all__ = ["login", "session", "User", "UserRepository"]
```

Keep the implementation in submodules, not in `__init__.py`.

### What NOT to do

- Don't create circular imports between `__init__.py` files. If A's
  `__init__.py` imports B and B's `__init__.py` imports A, you have a cycle.
- Don't put logic in `__init__.py` beyond re-exports — it runs on every import
  and makes cold start slow.

## Go

Go already enforces modularity (packages), so the refactor is usually about
splitting one big package into several smaller ones.

### Default target layout

```
internal/
  <feature>/
    feature.go         # public types and functions
    feature_test.go
    helper.go          # unexported helpers
```

### Package naming

- Short, lowercase, no underscores (`http`, `json`, `user`, `auth`).
- Avoid stutter: don't name it `user.User` — name it `user.User` only if the
  package is genuinely the user domain. Otherwise `model.User` or just
  `User` in package `user`.
- One concern per package. If your package doc comment lists three different
  things, split it.

### What to watch for

- Unexported identifiers (`lowercase`) can't cross package boundaries. If a
  helper is shared, either export it or duplicate (small helpers only).
- Circular package imports don't compile — Go catches them. Use this as a
  forcing function for clean boundaries.

## Rust

### Default target layout

```
src/
  lib.rs               # crate root, re-exports public API
  <feature>.rs         # or
  <feature>/
    mod.rs             # submodules
```

### `mod.rs` vs file-as-module

Both work; pick one per project and stay consistent. Newer style is
`feature.rs` + `feature/` directory, but `mod.rs` is fine for compatibility.

### `pub use` re-exports

```rust
// lib.rs
pub mod auth;
pub mod users;

pub use auth::{login, session};
```

### What to watch for

- Visibility cascades: making something `pub` to expose it to a sibling
  module is a smell. Prefer keeping things private and exposing via a small
  public API.
- `mod` declarations must match file structure. When splitting, update
  `mod` lists in `lib.rs`/`main.rs` first.

## Java

### Default target layout (Maven / Gradle)

```
src/main/java/com/<org>/<project>/
  <feature>/
    <Feature>Service.java
    <Feature>Controller.java
    <Feature>Repository.java
  shared/
    types/
  Application.java
```

### One class per file (mandatory)

Java enforces this at the compiler level. Split is usually mechanical.

### What to watch for

- Package-private (`default`) classes are useful for hiding implementation —
  don't promote everything to `public`.
- Spring / Jakarta annotations on classes mean moving the class also moves the
  bean wiring. Check `@ComponentScan` base packages.

## Cross-language rules

1. **Don't introduce new dependencies as part of a refactor.** If a move
   requires adding a new library, stop and confirm.
2. **Don't change behaviour, ever.** A refactor is definitionally
   behaviour-preserving. If you "while I'm here" add a feature or fix a bug,
   that's a separate commit on top of the refactor.
3. **Don't optimise during a split.** Move the code verbatim. Optimise in a
   later PR.
4. **Prefer many small files over few big ones**, but **don't over-fragment**.
   A 30-line file used in exactly one place doesn't need its own module —
   inline it.
5. **The directory structure should tell the story.** If a new contributor
   can't guess where a feature lives from the directory tree, the structure
   is wrong.