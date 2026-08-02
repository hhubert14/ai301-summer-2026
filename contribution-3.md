# Contribution 1: Use --config in ecosystem checks

**Contribution Number:** 3
**Student:** Hubert Huang
**Issue:** https://github.com/astral-sh/ruff/issues/10345
**Status:** Phase 2 Done

---

## Why I Chose This Issue

I chose issue #10345, "Use `--config` in ecosystem checks," because it's a well-scoped refactor in a project (Ruff) that I'm excited to contribute to as part of my CodePath AI301 capstone. The task is to move ecosystem-check overrides from temporary file mutation to CLI-level `--config` arguments.

This issue is a strong match for me because:
1. It's Python, which lines up with my experience building ML/data pipelines and infra tooling (including my current Capital One internship work on GenAI data guardrails), so I can ramp on the codebase quickly while still learning Ruff internals.
2. The scope is contained to two functions in a single file (`patch_config` and `to_ruff_args` in `ruff_ecosystem/projects.py`), which fits the 3–4 week window well — it's a targeted refactor, not a broad architectural rewrite.
3. There's substantial maintainer context already: the issue author (@zanieb) linked the exact code block to replace and outlined the intended approach, and a prior contributor (@hoel-bagard) already explored implementation details in PR #10436.
4. It's a good opportunity to learn how Ruff's configuration-resolution and CLI argument-parsing layers work, which is directly relevant to my interest in developer tooling and CLI-based systems.

From reading the thread, the current problem is that `ruff-ecosystem`'s `patch_config` context manager temporarily edits a target project's real config file (`.ruff.toml`, `ruff.toml`, or the `[tool.ruff]` table in `pyproject.toml`) and then reverts it. The requested direction is to pass equivalent overrides directly via `--config`.

I left a comment on the issue introducing myself and asking whether it's still open to take on, since the last activity (a linked PR attempt) was from over a year ago and no one is currently assigned.

---

## Understanding the Issue

### Problem Description

Ruff's ecosystem check tooling (`python/ruff-ecosystem/ruff_ecosystem/projects.py`) needs to apply a set of config overrides (e.g., always-on settings, preview-mode-specific settings) when running Ruff against third-party projects.

### Expected Behavior

Config overrides (`ALWAYS_CONFIG_OVERRIDES`, `self.always`, and either `self.when_preview` or `self.when_no_preview`) should be translated into one or more `--config key=value` arguments and passed directly to Ruff's CLI.

### Current Behavior

The `patch_config` context manager reads the target project's existing config file (preferring `.ruff.toml` > `ruff.toml` > `pyproject.toml`'s `[tool.ruff]` table), merges in the override dictionary (and in some cases deletes keys to represent "restore default"), writes the modified TOML to disk, runs checks, then restores the original file.

### Affected Components

- `python/ruff-ecosystem/ruff_ecosystem/projects.py`
  - `ConfigOverrides.patch_config()` — the context manager to be removed/replaced
  - `CheckOptions.to_ruff_args()` — where `--config` arguments should be appended instead
- Related prior attempt: PR #10436 ("replace `config_overrides.patch_config` by `config_overrides.to_ruff_args`"), useful for seeing what approach was already tried and where it may have stalled — especially around default/unset semantics.

### Open Question Flagged by a Prior Contributor

How to represent "restore this key to its default" (currently done by simply deleting the key from the TOML file) using `--config`, since TOML has no `null` literal to unset a value via a CLI-supplied key/value pair.

---

## Reproduction Process

### Environment Setup

To reproduce issue [astral-sh/ruff#10345](https://github.com/astral-sh/ruff/issues/10345), I set up a local Ruff development environment focused on the `ruff-ecosystem` tooling.

- Forked `astral-sh/ruff` and cloned my fork locally.
- Created a dedicated branch for investigation (before implementing fixes), e.g. `hh/repro-10345`.
- Installed project dependencies and dev tooling using Ruff’s contributor workflow (`CONTRIBUTING.md`), including Python and Rust toolchains needed to run ecosystem checks.
- Confirmed I could run relevant commands under `python/ruff-ecosystem/` and inspect behavior in `ruff_ecosystem/projects.py`.

**Challenges I faced + how I solved them**
1. **Toolchain/setup mismatch** (Python env + Rust build expectations): I resolved this by following Ruff’s contributor instructions step-by-step and using a clean virtual environment.
2. **Understanding where overrides are applied**: At first, it wasn’t obvious whether overrides were injected at CLI construction or file mutation time. I traced calls through `CheckOptions.to_ruff_args()` and `ConfigOverrides.patch_config()` to confirm current behavior.
3. **Reproducing safely without modifying real project configs permanently**: I used a disposable sample project config and inspected file contents before/after runs to verify temporary mutation behavior.

### Steps to Reproduce

1. Run ecosystem-check flow with config overrides enabled (preview/non-preview path) using a target project that has an existing Ruff config (`.ruff.toml`, `ruff.toml`, or `[tool.ruff]` in `pyproject.toml`).
2. Trace execution path in `python/ruff-ecosystem/ruff_ecosystem/projects.py`, specifically:
   - `ConfigOverrides.patch_config()` (context manager)
   - `CheckOptions.to_ruff_args()`
3. Observe behavior during execution:
   - Override values are merged into the on-disk config via `patch_config`.
   - CLI invocation does not fully represent those overrides as `--config key=value` arguments.
   - “Default restore” semantics currently rely on deleting keys in TOML rather than expressing unsets through CLI.

**Observed result:** The ecosystem check currently depends on temporary filesystem edits to project config files, rather than passing all overrides through `--config`, which is exactly the mismatch described in issue #10345.

### Reproduction Evidence

- **Commit showing reproduction:** [Link to commit in your fork]
- **Screenshots/logs:** [If applicable]
- **My findings:**
  1. Root behavior is confirmed: config overrides are applied primarily via temporary config-file mutation.
  2. Existing code already has a natural destination for CLI-level override expression (`to_ruff_args`), but the full override logic still lives in `patch_config`.
  3. The trickiest edge case is “reset key to default,” since deletion-from-file semantics don’t map 1:1 to TOML literals passed through `--config`.

---

## Solution Approach

### Analysis

The issue is caused by a design split: override computation exists, but application is done through `patch_config()` mutating target project config files. This was likely convenient for compatibility but creates fragility and diverges from the newer preferred interface (`--config`).

- **Current model:** compute overrides → edit TOML on disk → run Ruff → restore file.
- **Desired model:** compute overrides → append `--config key=value` args → run Ruff (no file mutation).

This shift improves determinism, avoids side effects in checked projects, and aligns with current Ruff CLI behavior.

### Proposed Solution

Refactor override application so `CheckOptions.to_ruff_args()` emits all needed `--config` arguments directly, and remove/retire file mutation logic from `ConfigOverrides.patch_config()`. Keep override precedence identical to current behavior (global always overrides + mode-specific overrides), and preserve behavior for both preview and non-preview runs.

For “restore default” keys, adopt a consistent explicit strategy based on maintainer guidance (or existing Ruff CLI capability if present), rather than relying on TOML key deletion in-place.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** `ruff-ecosystem` should apply check overrides via CLI `--config` arguments, not temporary edits to repository config files. Current behavior mutates files and restores them afterward, which issue #10345 asks to replace.

**Match:**
- Existing argument assembly pattern already exists in `CheckOptions.to_ruff_args()`.
- Existing override merge logic in `ConfigOverrides` can be reused as source-of-truth for key/value pairs.
- Prior attempt in PR #10436 provides context on what was tried and potential pitfalls (especially unset/default handling).

**Plan:**
1. Modify `python/ruff-ecosystem/ruff_ecosystem/projects.py` so `to_ruff_args()` appends override-derived `--config` args.
2. Add/adjust helper function(s) to serialize merged override dict into Ruff CLI-compatible `key=value` strings (including booleans/lists/strings formatting).
3. Deprecate/remove `patch_config()` usage from execution flow so checks no longer require temporary file mutation.
4. Handle default-restore edge case explicitly (based on maintainer-approved semantics).
5. Update tests to validate:
   - args now include expected `--config` entries,
   - no on-disk config rewriting occurs,
   - preview vs non-preview override behavior remains correct.

**Implement:** [Link to your branch/commits as you work]

**Review:**
- [ ] Followed Ruff contribution style and lint/test requirements.
- [ ] No unintended behavior change in override precedence.
- [ ] No residual file-mutation path in normal ecosystem check flow.
- [ ] Added/updated tests for both preview and non-preview cases.
- [ ] Commit messages are clear and scoped.
- [ ] Referenced issue #10345 in PR description and explained edge-case handling.

**Evaluate:**
- Run targeted tests for `ruff-ecosystem` argument construction.
- Run ecosystem-check command(s) before/after refactor and compare resulting effective behavior.
- Confirm generated CLI includes expected `--config` flags.
- Confirm target project config files remain unchanged throughout execution.
- Validate edge case(s) around default restore and document any intentional limitation if CLI cannot fully represent prior deletion semantics.

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
