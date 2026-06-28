# Contribution 2: Make docstring convention a global setting

**Contribution Number:** 2  
**Student:** Hubert Huang  
**Issue:** https://github.com/astral-sh/ruff/issues/9043  
**Status:** Phase III Complete  

---

## Why I Chose This Issue

Ruff is a widely-used Python linter built in Rust, and after my first contribution I wanted a second issue that pushed deeper into config-system design rather than copying an existing pattern. This one is about *settings ergonomics*: there are currently two separate places to declare a docstring convention, and the fix is to introduce a single global setting and reconcile it with the old one. That makes it a good vehicle for learning how user-facing options flow through a mature codebase (TOML → `Options` → resolved `Settings` → individual rules) and how a real project handles backward-compatible deprecation instead of just adding a flag.

I picked this specific issue because the design is already maintainer-approved: zanieb wrote "Adding a global setting is open for contribution," and MichaReiser laid out a concrete three-step plan (add the global option, implement merging logic that respects both the global and the pydocstyle-local setting, possibly ship it preview-only). Verification showed it is genuinely available — no assignee, no linked PR, no in-flight commits. The most recent activity is my own comment asking to take it on and `ntBre` removing the `help wanted` label shortly after; the label removal isn't a rejection (no comment, no assignment) but it is a signal worth surfacing in the PR description so reviewers know I'm aware. The scope is contained but slightly more substantial than a one-line change, which is exactly the step up I'm looking for: I get a pre-blessed design to anchor against, while still having to think through merge precedence, deprecation, and where a future consumer (the Pylint docstring rules) would read the setting.

---

## Understanding the Issue

### Problem Description

The docstring convention (Google, NumPy, or PEP 257) can only be set in one place today: `[tool.ruff.lint.pydocstyle]` → `convention`. That setting is scoped to the pydocstyle (`D`) rules. The problem is that other rules also care about the docstring convention — most concretely the Pylint docstring rules (e.g. the missing-return-doc rule proposed in #8843), and potentially a docstring formatter in the future. Because the convention lives *inside* the pydocstyle namespace, no other tool can read it; each consumer would need its own duplicate `convention` setting. There is no single, tool-wide place to say "this project uses Google-style docstrings."

### Expected Behavior

A single global `docstring-convention` option declared under `[tool.ruff]` (a sibling of `lint`/`format`, not nested under `pydocstyle`) that every docstring-aware part of Ruff reads from — the pydocstyle rules today, the Pylint docstring rules, and any future docstring tooling. The existing `[tool.ruff.lint.pydocstyle].convention` is deprecated in favor of it, with merging logic so existing configs keep working (a local pydocstyle value still takes effect / overrides the global one). Per MichaReiser's plan, this may ship as a preview-only change first.

### Current Behavior

The convention is pydocstyle-only and linter-scoped. If both pydocstyle and Pylint (or a future tool) need to know the convention, the user has to set it in two separate places, which is unintuitive and can silently drift or conflict — e.g. Google configured for pydocstyle but a different value, or nothing, for the Pylint rules. The maintainers have confirmed this is undesirable and that consolidating into a global setting is the intended direction.

### Affected Components

Confirmed by reading the source on `main` (commit `55d1f4404b`):

- **`crates/ruff_workspace/src/options.rs`** — the TOML-facing option definitions. The top-level `Options` struct (where a new `docstring-convention` field would be added) and `PydocstyleOptions` (line 3296), whose `convention` field at line 3389 is the current home of the setting, both live here.
- **`crates/ruff_workspace/src/configuration.rs`** — where options are resolved into settings (it calls `PydocstyleOptions::into_settings` and reads `pydocstyle.convention` at lines 1156-1163 to decide which `D` rules to disable). This is the natural home for the global-vs-local merge/precedence logic.
- **`crates/ruff_workspace/src/settings.rs`** — the resolved `Settings` that the linter actually consumes.
- **`crates/ruff_linter/src/rules/pydocstyle/settings.rs`** — defines the `Convention` enum (`Google`, `Numpy`, `Pep257`) and the pydocstyle `Settings { convention: Option<Convention>, ... }`. The new global option reuses this `Convention` type.
- **`ruff.schema.json`** — regenerated via `cargo dev generate-all` to reflect the new option.

---

## Reproduction Process

### Environment Setup

Developing on Linux 6.18 (WSL2) with `rustup`-managed Rust and the project's standard `cargo` workflow.

```sh
git clone https://github.com/hhubert14/ruff.git
cd ruff
cargo run --bin ruff -- --help
```

### Steps to Reproduce

The global `docstring-convention` option did not exist at the top level — any attempt to set it outside `[tool.ruff.lint.pydocstyle]` was rejected by serde's `deny_unknown_fields` with an `unknown field 'convention'` parse error.

---

## Solution Approach

### Analysis

The root cause is structural: the `convention` field was declared only on `PydocstyleOptions`, nested under `LintOptions.pydocstyle`. Promoting it to a global setting required adding a new field at the top-level `Options` struct and teaching the resolver in `configuration.rs` to consult both the global and local values.

### Proposed Solution

Add `docstring-convention` to the top-level `Options` struct, reusing the existing `Convention` enum. In `Configuration::into_settings`, resolve the effective convention before the two consumers (rule selection and runtime settings) run — local `lint.pydocstyle.convention` wins, global is the fallback.

---

## Implementation Notes

### Implementation Summary

The change adds a global `docstring-convention` setting under `[tool.ruff]` that acts as a fallback for the existing `[tool.ruff.lint.pydocstyle] convention`. When both are set, the more specific local setting takes precedence.

**Files modified:**

- **`crates/ruff_workspace/src/options.rs`** — Added `pub docstring_convention: Option<Convention>` to the top-level `Options` struct with full `#[option(...)]` documentation explaining the precedence behavior.

- **`crates/ruff_workspace/src/configuration.rs`** — Four changes:
  1. Added `use ruff_linter::rules::pydocstyle::settings::Convention` import
  2. Added `pub docstring_convention: Option<Convention>` field to the `Configuration` struct
  3. Wired it through `from_options` and `combine`
  4. Added precedence resolution in `into_settings`: if the global convention is set, get-or-insert a `PydocstyleOptions` and apply `pydocstyle.convention.or(Some(global_convention))` so the local value always wins

**Key precedence logic (3 lines):**
```rust
if let Some(global_convention) = self.docstring_convention {
    let pydocstyle = lint.pydocstyle.get_or_insert_with(PydocstyleOptions::default);
    pydocstyle.convention = pydocstyle.convention.or(Some(global_convention));
}
```

This runs before both rule selection (`as_rule_table`) and runtime settings (`into_settings`) so both consumers see a consistent resolved value.

### Code Changes

- **Branch:** https://github.com/hhubert14/ruff/tree/fix-issue-9043
- **Pull Request:** https://github.com/astral-sh/ruff/pull/26441

---

## Testing Strategy

Two new unit tests added to `crates/ruff_workspace/src/configuration.rs` inside the existing `mod tests` block:

- **`global_docstring_convention_applies_when_no_local`** — Sets only the global `docstring_convention` on `Configuration` with no `lint.pydocstyle` section. Verifies that `settings.linter.pydocstyle.convention()` returns the global value.

- **`local_docstring_convention_overrides_global`** — Sets both the global convention (`Google`) and a local `lint.pydocstyle.convention` (`Numpy`). Verifies that the local `Numpy` wins.

**Validation performed:**
- `cargo test -p ruff_workspace` — both new tests pass, all 26 existing tests pass
- `cargo clippy --workspace --all-targets --all-features -- -D warnings` — passes
- `RUFF_UPDATE_SCHEMA=1 cargo test` — passes (two pre-existing flaky `ty` CLI tests fail due to `ExecutableFileBusy` on WSL — unrelated to this change)
- `uvx prek run -a` — passes after auto-formatting
- `cargo dev generate-all` — JSON schema and docs regenerated

---

## Pull Request

**PR Link:** https://github.com/astral-sh/ruff/pull/26441

**Maintainer Feedback:**
- [Awaiting review]

**Status:** Awaiting review

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically — in your own words]

### Challenges Overcome

[What was hard and how you solved it — in your own words]

### What I'd Do Differently Next Time

[Reflection on your process — in your own words]

---

## Resources Used

- [Ruff CONTRIBUTING.md](https://github.com/astral-sh/ruff/blob/main/CONTRIBUTING.md)
- [Ruff AGENTS.md](https://github.com/astral-sh/ruff/blob/main/AGENTS.md)
- Issue #9043 discussion and MichaReiser's implementation plan
