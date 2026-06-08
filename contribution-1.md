# Contribution 1: Missing config option for `extend_external`

**Contribution Number:** 1  
**Student:** Hubert Huang  
**Issue:** https://github.com/astral-sh/ruff/issues/11921  
**Status:** Phase I Complete  

---

## Why I Chose This Issue

Ruff is a widely-used Python linter built in Rust, known for tight code quality and fast review turnaround. Contributing here is a good way to build experience with non-trivial Rust, config-system design, and the ecosystem-wide testing that large open-source projects rely on.

I picked this specific issue because Charlie Marsh (Ruff's creator) explicitly wrote "PR welcome" in the thread, the scope is contained (extending an existing config pattern rather than designing something new), and verification showed it is genuinely available: no assignee, no linked PR, and no in-flight commits from other contributors. As a first contribution, I'd rather land something small with a pre-approved design than fight maintainer pushback on a more ambitious change.

---

## Understanding the Issue

### Problem Description

Ruff's `external` config setting lets users declare lint codes from other tools so they aren't flagged as unknown. Sister settings like `select` and `ignore` have `extend-` variants (`extend-select`, `extend-ignore`) that let a child config add to the parent's list rather than overwrite it. `external` has no `extend-external`, so a project that inherits a shared base config can't add tool-specific external codes without redeclaring the whole list.

### Expected Behavior

Setting `extend-external = ["CODE"]` in a child `pyproject.toml` or `ruff.toml` appends `CODE` to whatever `external` codes were declared in the inherited base config.

### Current Behavior

There is no `extend-external` setting. Declaring `external` in the child config replaces the parent's list entirely, forcing the user to redeclare every code from the base.

### Affected Components

Ruff's config system: the lint settings where `external` is defined, and the workspace-level merger that handles `extend-`-style settings. Exact paths to be confirmed during reproduction, but I expect work in `crates/ruff_linter/` for the option definition and `crates/ruff_workspace/` for the merge logic. The TOML JSON schema (`ruff.schema.json`) is auto-generated from the Rust types, so it should update automatically. Documentation for the `external` setting will need a sibling `extend-external` entry. One open design question to address in the solution phase: MichaReiser was hesitant about adding `extend-` for every list setting and floated turning `external` into a table instead, but Charlie Marsh accepted the plain `extend-external` approach, which is the path I'll take.

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
