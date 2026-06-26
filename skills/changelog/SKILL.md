---
name: changelog
description: >-
  Produce librustzcash/Zebra-style CHANGELOG entries for the current branch (or a
  given PR/branch) by running ziff and writing them into the repo's CHANGELOG.md
  files; or, with --check, verify the existing entries against ziff and report
  discrepancies without writing. Use when the user asks to write/draft/produce or
  check/verify a changelog for a branch or PR, or turn a ziff diff into entries.
---

# Produce lrz-style changelogs with ziff

Run `ziff --changelog`, curate its draft into
[librustzcash](https://github.com/zcash/librustzcash)/Zebra-style entries, and
**write them into the repo's `CHANGELOG.md` files**. Needs `ziff` on `PATH` (plus
its prereqs: `cargo-public-api`, `jq`, a nightly toolchain).

## Target

- **No argument → the current branch** (the default, common case): produce
  changelogs for whatever the working tree is on, diffed against its branch point
  with `main`.
- An argument that is a **PR number/URL** or a **branch/ref** → use that instead.
- **`--check`** (with any target) → verify mode: report discrepancies against ziff and
  write nothing (see the `--check` section below). The fast pre-PR gate.

## Procedure

### 1. Resolve the diff range
- Current branch (no arg): nothing to resolve — ziff defaults to the branch point.
  Make sure the tree is **clean** first (commit work-in-progress), so ziff diffs
  the committed branch against its branch point rather than just uncommitted edits.
- PR `<N>`: `gh pr view <N> --json headRefName,headRefOid,baseRefName`. If it's
  checked out locally, treat it like the current branch; otherwise
  `git fetch <remote> pull/<N>/head` and pass explicit refs in step 2.

### 2. Run `ziff --changelog`
- Current branch: `ziff --changelog`.
- Explicit refs: `ziff --changelog $(git merge-base <base> <head>) <head>`.
- ziff runs a full `cargo public-api` build — run it where builds are fast (a
  remote build host for large workspaces). A flaky build can exit non-zero; rerun
  to confirm before treating it as a real failure.
- Output is per-crate `## <crate>` sections with `### Added/Changed/Removed` lists:
  paths already crate-relative, type members brace-grouped (`Type::{a, b}`), and
  over-wide groups already wrapped to a `Type:` header + 2-space sub-bullets.

### 3. Curate (ziff's output is a draft)
- **Sections**, in order: `### Breaking Changes` (if any), `### Added`,
  `### Changed`, `### Removed`, `### Fixed`. Keep ziff's crate-relative paths.
- **Periods**: no trailing period on pure-identifier / `impl … for …` / brace-group
  bullets; a period on prose bullets and parenthetical glosses.
- **`### Changed`**: ziff prints raw `old → new` signatures. Convert each to terse
  prose — a lead-in ending in `:` ("The following now take an additional
  `foo: Bar`:") with the affected items as 2-space sub-bullets — rather than
  dumping signatures. This is the main thing you add by hand; ziff can't infer it.
- **Fold over-granular items**: a struct variant's fields (e.g.
  `Foo::Bar::field`) belong inside the variant (`Foo::Bar { field }`), not as their
  own bullets.
- **Width**: wrap at ~100 chars with a 2-space hanging indent. Brace groups stay
  inline when they fit (ziff already breaks the over-wide ones).
- **No `X, with Y` joins** — list independent items as separate bullets.

### 4. Classify breaking changes
- ziff / `cargo public-api` count a new enum variant as merely *additive*, but
  adding a variant to a **non-`#[non_exhaustive]`** enum is breaking for downstream
  exhaustive `match`es → put it under `### Breaking Changes` and note that it needs
  a major version bump. Check the enum's attribute before deciding.

### 5. Write the entries to the files
- For each `## <crate>` ziff reports, open that crate's `CHANGELOG.md` and **insert
  the curated entries into its `[Unreleased]` section**, merging under the right
  subsections alongside anything already there — don't clobber existing entries, and
  skip anything already documented.
- Add a plain-language line to the workspace `CHANGELOG.md` for user-facing
  features; skip it for experimental / feature-gated work.
- **Leave the edits unstaged** — don't `git add` or commit them — so they show up
  in `git diff` for the user to review and stage selectively.
- Then show a `git diff --stat` of the touched changelogs and audit the result: no
  bare-identifier bullet has a trailing period, prose bullets do, every line ≤100
  chars.

## `--check` mode — verify, don't write

With `--check` (e.g. `/changelog --check`, optionally with a PR/branch arg), run the
diff (steps 1–2) but **report discrepancies and write nothing** — the fast pre-PR gate
that catches the "looks public but isn't" class of error a human can't eyeball:

1. From ziff's output, collect the public-API items: the backtick'd identifiers under
   each crate's `### Added` / `### Removed`, plus the old/new of `### Changed`.
2. Collect the changelog entries **this branch adds** — *not* the whole `[Unreleased]`
   section, which also holds other PRs' unreleased entries that ziff (diffing only this
   branch) won't report. Take the added (`+`) bullets from
   `git diff <branch-point>..HEAD -- '**/CHANGELOG.md' 'CHANGELOG.md'` (branch-point =
   `git merge-base main HEAD`, the same baseline ziff used), keeping only the ones that
   are a bare backtick'd identifier — ignore prose / behavioral bullets, since ziff only
   sees API signatures and those legitimately differ.
3. Report two lists and edit nothing:
   - **Listed but not public** — branch-added identifier bullets ziff doesn't report.
     Usually a path that isn't actually public (e.g. behind a `pub(crate)` module), or a
     line that belongs in prose rather than the identifier list.
   - **Public but undocumented** — items ziff reports that the branch didn't add.

   Both lists empty ⇒ the changelog's API entries match the public surface.

## Notes
- Library-crate changelogs follow librustzcash style (terse, code-pathed); the
  workspace `CHANGELOG.md` uses plain, user-facing descriptions.
- ziff's baseline is the branch point by default, so a stale local `main` or a
  branch that's behind upstream won't pollute the diff — no `--fetch` needed.
