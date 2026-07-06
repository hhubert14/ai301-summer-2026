# Contribution 1: Use --config in ecosystem checks

**Contribution Number:** 1
**Student:** Hubert Huang
**Issue:** https://github.com/astral-sh/ruff/issues/10345
**Status:** Phase I In Progress

---

## Why I Chose This Issue

I chose issue #10345, "Use `--config` in ecosystem checks," because it's a well-scoped refactor in a project (Ruff) that I'm excited to contribute to as part of my CodePath AI301 capstone. The task is to replace a workaround — where the ecosystem check tooling manually rewrites a project's `pyproject.toml`/`ruff.toml` file to apply config overrides, then restores it afterward — with Ruff's now-supported `--config` CLI flag, so overrides can be passed directly as command-line arguments instead of mutating files on disk.

This issue is a strong match for me because:
1. It's Python, which lines up with my experience building ML/data pipelines and infra tooling (including my current Capital One internship work on GenAI data guardrails), so I can ramp on the codebase quickly.
2. The scope is contained to two functions in a single file (`patch_config` and `to_ruff_args` in `ruff_ecosystem/projects.py`), which fits the 3–4 week window well — it's a targeted refactor, not a system redesign.
3. There's substantial maintainer context already: the issue author (@zanieb) linked the exact code block to replace and outlined the intended approach, and a prior contributor (@hoel-bagard) already surfaced the key open question (how to "unset"/restore a key to default using `--config`, since TOML has no `null`) along with a linked draft PR (#10436) I can learn from.
4. It's a good opportunity to learn how Ruff's configuration-resolution and CLI argument-parsing layers work, which is directly relevant to my interest in developer tooling and CLI-based systems.

From reading the thread, the current problem is that `ruff-ecosystem`'s `patch_config` context manager temporarily edits a target project's real config file (`.ruff.toml`, `ruff.toml`, or the `[tool.ruff]` table in `pyproject.toml`) to inject test-specific overrides, then restores the original file contents afterward. This is fragile (it mutates files on disk, has to carefully track and restore original state, and duplicates logic that already exists via the CLI) now that Ruff supports passing config overrides directly with `--config`. My contribution will replace the file-patching approach with building the appropriate `--config` arguments and passing them straight to the `ruff check` invocation in `to_ruff_args`.

I left a comment on the issue introducing myself and asking whether it's still open to take on, since the last activity (a linked PR attempt) was from over a year ago and no one is currently assigned.

---

## Understanding the Issue

### Problem Description

Ruff's ecosystem check tooling (`python/ruff-ecosystem/ruff_ecosystem/projects.py`) needs to apply a set of config overrides (e.g., always-on settings, preview-mode-specific settings) when running Ruff against third-party "ecosystem" projects for CI comparison checks. Currently it does this by directly editing the target project's config file on disk (`patch_config`), applying the overrides, running the check, and then restoring the original file. Now that Ruff has a `--config` CLI flag that accepts inline config overrides, this file-patching workaround is unnecessary and should be replaced.

### Expected Behavior

Config overrides (`ALWAYS_CONFIG_OVERRIDES`, `self.always`, and either `self.when_preview` or `self.when_no_preview`) should be translated into one or more `--config key=value` arguments and passed directly to the `ruff check` subprocess call — with no modification of any file in the target project's directory, and no need to save/restore original file contents.

### Current Behavior

The `patch_config` context manager reads the target project's existing config file (preferring `.ruff.toml` > `ruff.toml` > `pyproject.toml`'s `[tool.ruff]` table), merges in the override dictionary (removing keys entirely when the override value is `None`, to "restore to default"), writes the modified TOML back to disk, yields control to the caller to run the check, and then restores the original file contents (or deletes the file if it didn't previously exist) in a `finally` block.

### Affected Components

- `python/ruff-ecosystem/ruff_ecosystem/projects.py`
  - `ConfigOverrides.patch_config()` — the context manager to be removed/replaced
  - `CheckOptions.to_ruff_args()` — where `--config` arguments should be appended instead
- Related prior attempt: PR #10436 ("replace `config_overrides.patch_config` by `config_overrides.to_ruff_args`"), useful for seeing what approach was already tried and where it may have stalled — worth reviewing before starting implementation.

### Open Question Flagged by a Prior Contributor

How to represent "restore this key to its default" (currently done by simply deleting the key from the TOML file) using `--config`, since TOML has no `null` literal to unset a value via a CLI-supplied override string. This will need to be resolved early in Phase II, likely by checking how Ruff's `--config` argument parser (added by @AlexWaygood) handles unset/default values internally.

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

- Original issue thread: https://github.com/astral-sh/ruff/issues/10345
- Linked prior attempt: PR #10436
- Ruff `CONTRIBUTING.md` (to review before local setup in Phase II)