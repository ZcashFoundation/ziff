---
name: changelog
description: >-
  Produce librustzcash/Zebra-style CHANGELOG entries for a pull request (or
  branch) by running ziff and curating its draft. Use when the user asks to
  write/draft/produce a changelog for a PR, update CHANGELOG.md for a branch's
  changes, or turn a ziff diff into changelog entries.
---

# Produce lrz-style changelogs for a PR with ziff

Turn a PR's public-API and dependency changes into curated,
[librustzcash](https://github.com/zcash/librustzcash)/Zebra-style `CHANGELOG.md`
entries. `ziff` generates the raw draft; you curate it. Needs `ziff` on `PATH`
(plus its prereqs: `cargo-public-api`, `jq`, a nightly toolchain).

## Input

A PR number/URL, or a branch/ref. Default to the current branch if none is given.

## Procedure

### 1. Locate the change
- PR: `gh pr view <N> --json headRefName,headRefOid,baseRefName`. Fetch the head
  if it isn't local: `git fetch <remote> pull/<N>/head`.
- You want the PR's *own* changes — the diff against its **branch point** with the
  base (usually `main`), not against the base tip.

### 2. Run `ziff --changelog`
- On the PR branch with no args, ziff defaults to the branch point:
  `ziff --changelog`. Or be explicit: `ziff --changelog $(git merge-base <base> <head>) <head>`.
- ziff runs a full `cargo public-api` build — run it where builds are fast (a
  remote build host for large workspaces). A flaky build can exit non-zero; rerun
  to confirm before treating it as a real failure.
- Output is per-crate `## <crate>` sections with `### Added/Changed/Removed` lists:
  paths already crate-relative, type members brace-grouped (`Type::{a, b}`), and
  over-wide groups already wrapped to a `Type:` header + 2-space sub-bullets.

### 3. Curate into final lrz style
ziff's output is a **draft**. Apply these conventions:
- **Sections**, in order: `### Breaking Changes` (if any), `### Added`,
  `### Changed`, `### Removed`, `### Fixed`. Keep ziff's crate-relative paths.
- **Periods**: no trailing period on pure-identifier / `impl … for …` / brace-group
  bullets; a period on prose bullets and parenthetical glosses.
- **`### Changed`**: ziff prints raw `old → new` signatures. Convert each to terse
  prose — e.g. a lead-in ending in `:` ("The following now take an additional
  `foo: Bar`:") with the affected items as 2-space sub-bullets — rather than
  dumping signatures. This is the main thing you add by hand; ziff can't infer it.
- **Width**: wrap at ~100 chars with a 2-space hanging indent. Brace groups stay
  inline when they fit (ziff already breaks the over-wide ones).
- **No `X, with Y` joins** — list independent items as separate bullets.
- **Combine, don't clobber**: merge into the existing `[Unreleased]` section,
  folding new entries under the right subsections alongside what's already there.

### 4. Classify breaking changes
- ziff / `cargo public-api` count a new enum variant as merely *additive*, but
  adding a variant to a **non-`#[non_exhaustive]`** enum is breaking for downstream
  exhaustive `match`es → put it under `### Breaking Changes` and note that it needs
  a major version bump. Check the enum's attribute before deciding.

### 5. Apply and verify
- Write the entries into each crate's `CHANGELOG.md` `[Unreleased]` section. Add a
  plain-language line to the workspace `CHANGELOG.md` for user-facing features;
  skip it for experimental / feature-gated work.
- Audit the result: no bare-identifier bullet has a trailing period, prose bullets
  do, and every line is ≤100 chars.

## Notes
- Library-crate changelogs follow librustzcash style (terse, code-pathed); the
  workspace `CHANGELOG.md` uses plain, user-facing descriptions.
- ziff's baseline is the branch point by default, so a stale local `main` or a
  branch that's behind upstream won't pollute the diff — no `--fetch` needed.
