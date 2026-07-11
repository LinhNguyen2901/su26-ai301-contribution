# Contribution 2: `Programming Exercise`: Bonus Points: 1e-30 is considered a valid value

**Contribution Number:** 2
**Student:** Linh Nguyen
**Issue:** https://github.com/ls1intum/Artemis/issues/12451
**Status:** Phase I Complete

---

## Why I Chose This Issue

I chose this issue because it's a small, well-scoped bug with a clear description and no existing comments, which made it approachable as a first contribution. I wanted to start with something narrow enough to fully understand end-to-end rather than a large feature, while still working on a project used in a real production setting (Artemis is an educational platform built and used by TUM, Technical University of Munich, for teaching programming courses). This issue also touches form validation, which is a good entry point for learning Artemis's Angular/TypeScript client codebase, and it gives me a chance to practice finding and reusing an existing pattern (the validation already used on the regular Points field) rather than designing something from scratch.

---

## Understanding the Issue

### Problem Description

When creating or editing a programming exercise in Artemis, the Bonus Points field does not validate its input the same way the regular Points field does. As a result, invalid values (e.g., something like `1e-3` or `1e-30`) are accepted as a bonus point value when they should be rejected.

### Expected Behavior

The Bonus Points field should apply the same validation rules as the Points field, rejecting the same invalid values (e.g., overly precise or scientific-notation values that aren't valid point amounts).

### Current Behavior

A value like `1e-3`/`1e-30` is accepted as valid input in the Bonus Points field, even though the equivalent value would be rejected in the Points field.

### Affected Components

Likely the programming exercise creation/edit form in the Angular client (form validation logic for the exercise's points/bonus points fields).

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

- Artemis Contribution Guidelines: https://github.com/ls1intum/Artemis/blob/develop/CONTRIBUTING.md
- Artemis Developer Setup Guide: https://docs.artemis.tum.de/developer/setup
- [Tutorial or Stack Overflow post that helped]
- [GitHub issues or discussions that helped]