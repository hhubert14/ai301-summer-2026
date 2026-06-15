# Contribution 2: Make docstring convention a global setting

**Contribution Number:** 2  
**Student:** Hubert Huang  
**Issue:** https://github.com/astral-sh/ruff/issues/9043  
**Status:** Phase I In Progress  

---

## Why I Chose This Issue

Ruff is a widely-used Python linter built in Rust, and after my first contribution I wanted a second issue that pushed deeper into config-system design rather than copying an existing pattern. This one is about *settings ergonomics*: there are currently two separate places to declare a docstring convention, and the fix is to introduce a single global setting and reconcile it with the old one. That makes it a good vehicle for learning how user-facing options flow through a mature codebase (TOML → `Options` → resolved `Settings` → individual rules) and how a real project handles backward-compatible deprecation instead of just adding a flag.

I picked this specific issue because the design is already maintainer-approved: zanieb wrote "Adding a global setting is open for contribution," and MichaReiser laid out a concrete three-step plan (add the global option, implement merging logic that respects both the global and the pydocstyle-local setting, possibly ship it preview-only). Verification showed it is genuinely available — no assignee, no linked PR, no in-flight commits, and the last activity was September 2024. The scope is contained but slightly more substantial than a one-line change, which is exactly the step up I'm looking for: I get a pre-blessed design to anchor against, while still having to think through merge precedence, deprecation, and where a future consumer (the Pylint docstring rules) would read the setting.

---

## Understanding the Issue

### Problem Description

The docstring convention (Google, NumPy, or PEP 257) can only be set in one place today: `[tool.ruff.lint.pydocstyle]` → `convention`. That setting is scoped to the pydocstyle (`D`) rules. The problem is that other rules also care about the docstring convention — most concretely the Pylint docstring rules (e.g. the missing-return-doc rule proposed in #8843), and potentially a docstring formatter in the future. Because the convention lives *inside* the pydocstyle namespace, no other tool can read it; each consumer would need its own duplicate `convention` setting. There is no single, tool-wide place to say "this project uses Google-style docstrings."

### Expected Behavior

A single global `docstring-convention` option declared under `[tool.ruff]` (a sibling of `lint`/`format`, not nested under `pydocstyle`) that every docstring-aware part of Ruff reads from — the pydocstyle rules today, the Pylint docstring rules, and any future docstring tooling. The existing `[tool.ruff.lint.pydocstyle].convention` is deprecated in favor of it, with merging logic so existing configs keep working (a local pydocstyle value still takes effect / overrides the global one). Per MichaReiser's plan, this may ship as a preview-only change first.

### Current Behavior

The convention is pydocstyle-only and linter-scoped. If both pydocstyle and Pylint (or a future tool) need to know the convention, the user has to set it in two separate places, which is unintuitive and can silently drift or conflict — e.g. Google configured for pydocstyle but a different value, or nothing, for the Pylint rules. The maintainers have confirmed this is undesirable and that consolidating into a global setting is the intended direction.

### Affected Components

Confirmed by reading the source on `main`:

- **`crates/ruff_workspace/src/options.rs`** — the TOML-facing option definitions. The top-level `Options` struct (where a new `docstring-convention` field would be added) and `PydocstyleOptions`, whose `convention` field is the current home of the setting, both live here.
- **`crates/ruff_workspace/src/configuration.rs`** — where options are resolved into settings (it calls `PydocstyleOptions::into_settings`). This is the natural home for the global-vs-local merge/precedence logic.
- **`crates/ruff_workspace/src/settings.rs`** — the resolved `Settings` that the linter actually consumes.
- **`crates/ruff_linter/src/rules/pydocstyle/settings.rs`** — defines the `Convention` enum (`Google`, `Numpy`, `Pep257`) and the pydocstyle `Settings { convention: Option<Convention>, ... }`. The new global option would reuse this `Convention` type.
- **Pylint docstring rules** (`crates/ruff_linter/src/rules/pylint/`) — the second consumer; wiring these to read the resolved convention is what demonstrates the global setting's value.
- **`ruff.schema.json`** — the JSON schema is generated from the Rust option types, so it should regenerate once the new option is added (CI checks this is in sync).
- **Documentation (`docs/`)** — per the proposal, only the new global setting should be documented going forward, with the pydocstyle-local one marked deprecated.

---

## Reproduction Process

### Environment Setup

[Notes on setting up your local development environment - challenges you faced, how you solved them]

### Steps to Reproduce

1. [Step 1]
2. [Step 2]
3. [Observed result]

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:** [What you discovered during reproduction]

---

## Solution Approach

### Analysis

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** [Restate the problem]

**Match:** [What similar patterns/solutions exist in the codebase?]

**Plan:** [Step-by-step implementation plan]
1. [Modify file X to do Y]
2. [Add function Z]
3. [Update tests]

**Implement:** [Link to your branch/commits as you work]

**Review:** [Self-review checklist - does it follow the project's contribution guidelines?]

**Evaluate:** [How will you verify it works?]

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
