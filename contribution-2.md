# Contribution 2: Make docstring convention a global setting

**Contribution Number:** 2  
**Student:** Hubert Huang  
**Issue:** https://github.com/astral-sh/ruff/issues/9043  
**Status:** Phase II Complete  

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
- **`crates/ruff_workspace/src/configuration.rs`** — where options are resolved into settings (it calls `PydocstyleOptions::into_settings` and reads `pydocstyle.convention` at lines 1156-1163 to disable convention-incompatible rules). This is the natural home for the global-vs-local merge/precedence logic.
- **`crates/ruff_workspace/src/settings.rs`** — the resolved `Settings` that the linter actually consumes.
- **`crates/ruff_linter/src/rules/pydocstyle/settings.rs`** — defines the `Convention` enum (`Google`, `Numpy`, `Pep257`) and the pydocstyle `Settings { convention: Option<Convention>, ... }`. The new global option would reuse this `Convention` type.
- **Pylint docstring rules** (`crates/ruff_linter/src/rules/pylint/`) — the second consumer; wiring these to read the resolved convention is what demonstrates the global setting's value. Note: `crates/ruff_linter/src/rules/pylint/settings.rs` currently has no `convention` field, so this is forward-looking rather than fixing an already-duplicated setting.
- **`ruff.schema.json`** — the JSON schema is generated from the Rust option types via `cargo dev generate-all`, so it should regenerate once the new option is added (CI checks this is in sync).
- **Documentation (`docs/`)** — per the proposal, only the new global setting should be documented going forward, with the pydocstyle-local one marked deprecated.

---

## Reproduction Process

### Environment Setup

I'm developing on Linux 6.18 (WSL2) with `rustup`-managed Rust and the project's standard `cargo` workflow. No special bootstrapping was needed beyond cloning my fork and letting `cargo` resolve the workspace:

```sh
git clone https://github.com/hhubert14/ruff.git
cd ruff
# Debug builds are fine for development — AGENTS.md explicitly recommends them.
cargo run --bin ruff -- --help
```

The first `cargo run` builds the full workspace from scratch (a few minutes), but after that incremental builds are fast. I did **not** need to install Python locally to reproduce this particular issue, because the bug is in TOML parsing: the option I want to set is rejected before any Python is ever inspected.

A few things worth noting for anyone repeating the setup:

- **Don't create reproduction `.py` files inside the Ruff checkout.** Per `AGENTS.md`, the root `pyproject.toml`'s `requires-python = ">=3.7"` is picked up by sibling tools and pins their Python version inference. I put all repro files under `/tmp` to avoid this.
- The debug binary lives at `target/debug/ruff` after a build; `cargo run --bin ruff -- <args>` is the most convenient entry point.

### Steps to Reproduce

The goal of the reproduction is to demonstrate two things: (a) the current `[tool.ruff.lint.pydocstyle]` placement works, and (b) the same option declared anywhere outside that namespace is rejected by the TOML parser — there is no global escape hatch.

1. Create a small target Python file at `/tmp/test_convention.py`:

   ```python
   def hello(name):
       """Says hello.

       Args:
           name: Person's name.
       """
       return f"Hello, {name}"
   ```

2. **Baseline: nested setting works.** Create `/tmp/ruff_nested.toml`:

   ```toml
   [lint]
   select = ["D"]

   [lint.pydocstyle]
   convention = "google"
   ```

   Then run:

   ```sh
   cargo run --bin ruff -- check --config /tmp/ruff_nested.toml --no-cache /tmp/test_convention.py
   ```

   **Expected:** ruff applies the Google convention (disabling `D203`/`D204`/etc.) and reports only Google-applicable diagnostics.
   **Actual:** matches — `D100 Missing docstring in public module`, exit code 1 from ruff but `cargo run` reports success on the build side. ✅ Setting works in this location.

3. **Attempt the global setting under `[lint]`.** Create `/tmp/ruff_lint_global.toml`:

   ```toml
   [lint]
   select = ["D"]
   convention = "google"
   ```

   Run the same `ruff check` command pointing at this config.

   **Expected (per the issue):** ruff would honor a `[lint]`-level convention shared across all docstring-aware lint rules.
   **Actual:**

   ```
   ruff failed
     Cause: Failed to load configuration `/tmp/ruff_lint_global.toml`
     Cause: Failed to parse /tmp/ruff_lint_global.toml
     Cause: TOML parse error at line 3, column 1
       |
     3 | convention = "google"
       | ^^^^^^^^^^
     unknown field `convention`, expected one of `allowed-confusables`, `dummy-variable-rgx`,
     `extend-ignore`, `extend-select`, ..., `pydocstyle`, `pylint`, ...
   ```

   ❌ No global setting available at the lint scope.

4. **Attempt the global setting at the top level (`[tool.ruff]`).** Create `/tmp/ruff_top.toml`:

   ```toml
   convention = "google"

   [lint]
   select = ["D"]
   ```

   Run the same `ruff check` command.

   **Expected (per the issue):** ruff would honor a top-level `convention` shared across the entire tool (linter + future formatter + future Pylint docstring rules).
   **Actual:**

   ```
   ruff failed
     Cause: Failed to load configuration `/tmp/ruff_top.toml`
     Cause: Failed to parse /tmp/ruff_top.toml
     Cause: TOML parse error at line 1, column 1
       |
     1 | convention = "google"
       | ^
     unknown field `convention`
   ```

   ❌ No global setting available at the project scope either.

5. **Source confirmation.** A grep across the workspace confirms the parser's rejection isn't a hidden-feature situation:

   ```sh
   grep -rn "docstring-convention\|docstring_convention" crates/
   ```

   The only matches are two test-function names (`select_docstring_convention_override`, `check_docstring_conventions_overrides`); neither defines or wires up a global option. The single `pub convention: Option<Convention>` declaration in `crates/ruff_workspace/src/options.rs:3389` belongs to `PydocstyleOptions`, confirming the field is reachable only via the nested path.

### Reproduction Evidence

- **Working branch:** https://github.com/hhubert14/ruff/tree/fix-issue-9043
- **Reproduced against:** upstream `main` at commit `55d1f4404b` (June 2026).
- **Logs:** the full `unknown field 'convention'` parse errors shown in steps 3 and 4 are reproducible verbatim — they come straight from serde's `deny_unknown_fields` on `LintOptions` / `Options`.
- **My findings:** The issue is *unambiguously* still present; the global setting has never been added. The reproduction also incidentally clarifies one framing point in the original issue: the *current* duplication concern is forward-looking, because Pylint's own `convention` setting (referenced in #8843) was not merged in its proposed form — `crates/ruff_linter/src/rules/pylint/settings.rs` has no `convention` field today. So the change defends against a future second copy more than it eliminates an existing one. That's worth being honest about in the PR description.

---

## Solution Approach

### Analysis

The root cause is purely structural: the `convention` field is declared only on `PydocstyleOptions` (`crates/ruff_workspace/src/options.rs:3389`), and that struct is in turn nested under `LintOptions.pydocstyle`. serde's `deny_unknown_fields` on both `Options` and `LintOptions` enforces this — any sibling-level `convention` key is rejected at parse time. Inside the resolver (`crates/ruff_workspace/src/configuration.rs:1156-1163`), the code reads `self.pydocstyle.as_ref().and_then(|p| p.convention)` to decide which `D` rules to disable; nothing else in the workspace looks at the convention. So promoting it cleanly requires both (a) adding a new field at the right scope and (b) teaching the resolver to consult both locations.

### Proposed Solution

Add a top-level `docstring-convention` option to `Options` in `crates/ruff_workspace/src/options.rs`, reusing the existing `Convention` enum from `crates/ruff_linter/src/rules/pydocstyle/settings.rs`. Thread the value through configuration resolution so that, when ruff produces the final `pydocstyle::settings::Settings`, the resolved convention is: **local `[lint.pydocstyle].convention` if explicitly set, otherwise the global `[tool.ruff].docstring-convention`, otherwise `None` (the current default).** Keep the old field working — no breaking change. Gate the new option behind `preview` per MichaReiser's suggestion so the design can be revised before stabilization.

The existing precedent for "local wins, fall back to global" is `PycodestyleOptions::into_settings(self, global_line_length: LineLength)` at `options.rs:3280-3287`, which already takes a global `line-length` parameter and falls back to it when the pycodestyle-local override is absent. I'll model `PydocstyleOptions::into_settings` on that signature.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** The docstring convention is currently locked inside the `[tool.ruff.lint.pydocstyle]` namespace, which prevents other docstring-aware code (Pylint rules, future formatter) from reading it without users declaring it twice. Make it a global option under `[tool.ruff]` while keeping the existing nested option as a (deprecated, locally-overriding) fallback.

**Match:**
- **Global-fallback pattern**: `PycodestyleOptions::into_settings(self, global_line_length: LineLength)` (`crates/ruff_workspace/src/options.rs:3280-3287`) is the closest existing precedent — a tool-local override that falls back to a global value. I'll mirror this for `PydocstyleOptions::into_settings`.
- **Preview gating**: many recently-added options in `options.rs` use `#[option(...)]` plus a preview check in `configuration.rs` to keep semantics gated. I'll search for an existing preview-only option as a template before writing my own.
- **Existing convention-consumption site**: `crates/ruff_workspace/src/configuration.rs:1156-1163` is the only place that reads `pydocstyle.convention` today. The same resolved value should flow through here.

**Plan:**
1. **Add the global option.** In `crates/ruff_workspace/src/options.rs`, add `pub docstring_convention: Option<Convention>` to the top-level `Options` struct (sibling of `lint`/`format`, kebab-cased as `docstring-convention` in TOML), with the same `#[option(...)]` docs as the existing pydocstyle one, plus a note that it's global. Re-export or import the `Convention` type as needed.
2. **Resolve precedence.** Update `PydocstyleOptions::into_settings` to accept the global value as a parameter (`global_docstring_convention: Option<Convention>`), returning `convention.or(global_docstring_convention)` — local wins, falls back to global. Update the call site in `configuration.rs` to pass the new argument.
3. **Update the rule-disabling logic.** In `configuration.rs:1156-1163`, read the resolved convention (from the new global value if no local was set) rather than only `pydocstyle.convention`. This is what unlocks the rest of the codebase reading it.
4. **Preview gating.** Per MichaReiser, ship behind the `preview` flag. When `preview = false` and only the global option is set, either ignore it with a warning or treat it as an error — TBD based on how other preview-only options behave; I'll match the existing pattern.
5. **Schema + docs regen.** Run `cargo dev generate-all` per `AGENTS.md` to regenerate `ruff.schema.json`, the option-reference docs, and any CLI reference that lists the option. Verify CI's diff check is clean.
6. **Out of scope for this PR (intentional):** *consuming* the new global setting from Pylint rules. The issue's value is unlocked by step 1 — once the option exists and is reachable via the resolved `Settings`, a future PR can wire up Pylint or formatter consumers. Bundling that here would expand the diff and the review surface; the issue text explicitly frames consumption as a separate, follow-up benefit. I'll note this in the PR description so reviewers don't expect it.

**Implement:** Branch — https://github.com/hhubert14/ruff/tree/fix-issue-9043 (work to follow in Phase III).

**Review:**
- Read `CONTRIBUTING.md` and `AGENTS.md` end-to-end before opening the PR (already done — note in particular: PR title format is plain (no `[ty]` prefix since this is a Ruff change), narrow visibility by default, no local imports inside functions, prefer let-chains, run `uvx prek` on touched files before committing).
- Sanity-check no `panic!`/`unwrap`/`unreachable!` introduced — the `Option<Convention>` plumbing is naturally panic-free, but `cargo dev generate-all` sometimes touches generated code I should re-read.
- Confirm the new option's doc comment doesn't accidentally reference the old nested location as the recommended path.

**Evaluate:**
- **Unit tests** in `crates/ruff_workspace/src/configuration.rs` next to the existing `select_docstring_convention_override` test (lines 2166+):
  - Global-only set → resolved convention is the global one.
  - Local-only set (existing behavior) → resolved convention is the local one (regression guard).
  - Both set → local wins; the global is ignored.
  - Neither set → `None`, default behavior unchanged.
- **Integration tests** in `crates/ruff/tests/integration_test.rs` near `check_docstring_conventions_overrides`: a TOML fixture with the new global option and an end-to-end `ruff check` invocation, asserting the right `D` rules are toggled off.
- **Snapshot updates**: run with `INSTA_FORCE_PASS=1 INSTA_UPDATE=always MDTEST_UPDATE_SNAPSHOTS=1` per `AGENTS.md`, then diff-review every changed snapshot before committing — particularly the options reference documentation, which will mention the new field.
- **Manual TOML repro**: re-run the four cases from "Steps to Reproduce" above on my branch. Cases 3 and 4 (the previously-failing ones) should now succeed; case 2 (nested) should be unchanged; a fifth case where both are set should resolve to the local value.

---

## Testing Strategy

### Unit Tests

- [ ] Test case 1: [Description]
- [ ] Test case 2: [Description]
- [ ] Test case 3: [Description]

### Integration Tests

- [ ] Integration scenario 1
- [ ] Integration scenario 2

### Manual Testing

[What you tested manually and results]

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Week [Y] Progress

[Continue documenting as you work]

### Code Changes

- **Files modified:** [List]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description - much of the content above can be adapted]

**Maintainer Feedback:**
- [Date]: [Summary of feedback received]
- [Date]: [How you addressed it]

**Status:** [Awaiting review / Iterating / Approved / Merged]

---

## Learnings & Reflections

### Technical Skills Gained

[What you learned technically]

### Challenges Overcome

[What was hard and how you solved it]

### What I'd Do Differently Next Time

[Reflection on your process]

---

## Resources Used

- [Link to helpful documentation]
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]
