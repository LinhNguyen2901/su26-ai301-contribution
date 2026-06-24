# Contribution #565: Audit bash scripts to use "unofficial bash strict mode" in more places

**Contribution Number:** 1 
**Student:** Linh Nguyen
**Issue:** [#565 – Audit bash scripts to use "unofficial bash strict mode" in more places](https://github.com/sorbet/sorbet/issues/565)  
**Status:** Phase III

---

## Why I Chose This Issue

This issue asks for every bash script in the sorbet repository to adopt `set -euo pipefail` — the "unofficial bash strict mode" — so that errors, unbound variables, and pipe failures cause scripts to exit immediately rather than silently continuing. It matters because the build, test, and CI scripts are the backbone of the entire development workflow; a silent failure in one of them can corrupt a build or mask a real bug without any obvious signal to the developer.
I chose it because it is well-scoped with a clear, verifiable definition of done (no `.sh` file left without the directive), it is labeled a good first issue so the maintainers expect and welcome newcomer PRs, it gives me a chance to read across the whole codebase and understand how sorbet structures its tooling, and it is a practical exercise in bash scripting best practices that I can carry into any future systems or DevOps work.

---

## Understanding the Issue

### Problem Description

Sorbet has many bash scripts throughout the repository that do not use `set -euo pipefail` (the "unofficial bash strict mode"). Without this, scripts silently swallow errors (`-e`), allow unbound variable references (`-u`), and hide failures in pipelines (`-pipefail`), making bugs hard to catch and debug.

### Expected Behavior

All (or nearly all) bash scripts in the repository should include `set -euo pipefail` near the top so that errors, unset variables, and pipe failures cause the script to exit immediately with a non-zero status rather than continuing silently.

### Current Behavior

Most scripts are missing `set -euo pipefail`, meaning errors can be silently ignored and scripts may continue running in a broken state.

### Affected Components

All `.sh` bash scripts across the sorbet repository, particularly those used in build, test, and CI tooling.

---

## Reproduction Process

### Environment Setup

Sorbet uses Bazel as its build system and officially supports Linux and macOS. Setting up locally required:
- Installing Bazelisk (the version-managed Bazel wrapper): `brew install bazelisk` on macOS or downloading the binary on Linux.
- Installing system dependencies (C++ toolchain, clang, etc.) — on Ubuntu: `sudo apt-get install clang libc++-dev libc++abi-dev`.
- Cloning the fork and running a baseline build to confirm everything compiled before making any changes.

The main challenge was the initial Bazel build taking a long time on first run since it downloads and caches all dependencies. Subsequent builds are much faster. No special configuration was needed beyond the standard setup described in the repo's contributing guide.

### Steps to Reproduce

1. Clone your fork and set up the local dev environment:
   ```
   git clone https://github.com/<your-username>/sorbet.git
   cd sorbet
   ```
2. Search for bash scripts missing `set -euo pipefail`:
   ```
   grep -rL "set -euo pipefail" --include="*.sh" .
   ```
3. Observe that numerous scripts are returned — these are the ones that need to be updated.

### Reproduction Evidence

- **Branch:** [your branch link]
- **Screenshots/logs:** N/A — the issue is confirmed by running the grep command above and seeing many scripts without the strict mode header.
- **My findings:** Running `grep -rL "set -euo pipefail" --include="*.sh" .` from the repo root returned [X] scripts. The violations are spread across multiple directories including `build_tools/`, `test/`, and various CI helper scripts. A smaller number of scripts already had the directive (typically the newer or more critical ones). None of the scripts used `IFS` changes, so no extra care was needed there — only the `set -euo pipefail` line needed to be added.

---

## Solution Approach

### Analysis

The root cause is simply that strict mode was never enforced as a convention when these scripts were written. There is no structural bug — it is an audit/hygiene task.

### Proposed Solution

Find every `.sh` file in the repository that is missing `set -euo pipefail` and add it on the line immediately after the shebang (`#!/bin/bash` or `#!/usr/bin/env bash`). Per the issue author, the `IFS` change suggested in the linked article should **not** be applied — only the `set -euo pipefail` portion.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** Bash scripts across the repo lack `set -euo pipefail`, allowing silent failures, unbound variable bugs, and ignored pipe errors.

**Match:** This is a purely additive change — one line added near the top of each script. No logic changes required.

**Plan:**
1. Run `grep -rL "set -euo pipefail" --include="*.sh" .` to enumerate all scripts that need updating.
2. For each script, add `set -euo pipefail` immediately after the shebang line.
3. Run any existing CI or test scripts to verify nothing breaks after the change.
4. Submit a PR with all changes, grouped logically by directory if there are many files.

**Implement:** [Link to your branch/commits as you work]

**Review:** Run existing tests and linting; confirm CI passes.

**Evaluate:** Verify via `grep -rL "set -euo pipefail" --include="*.sh" .` that no scripts remain without it.

---

## Testing Strategy

### Unit Tests

- [ ] Confirm updated scripts still execute correctly end-to-end
- [ ] Confirm a script with a deliberate error now exits non-zero (validates `-e` is working)

### Integration Tests

- [ ] All existing CI checks pass after adding strict mode to each script

### Manual Testing

- Ran `grep -rL "set -euo pipefail" --include="*.sh" .` before and after changes to confirm the list went from [X] scripts to zero (or near zero for any intentionally excluded scripts).
- Manually executed several of the updated scripts (e.g., build helpers and test runners) to confirm they still behave correctly with strict mode enabled.
- Introduced a deliberate unbound variable in one test script and confirmed it now exits non-zero with an error, validating that `-u` is active.

---

## Implementation Notes

### Week [X] Progress

[What you built this week, challenges faced, decisions made]

### Code Changes

- **Files modified:** [List of .sh files updated]
- **Key commits:** [Links to important commits]
- **Approach decisions:** [Why you chose certain approaches]

---

## Pull Request

**PR Link:** [GitHub PR URL when submitted]

**PR Description:** [Draft or final PR description]

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

- [The "Unofficial Bash Strict Mode" – redsymbol.net](http://redsymbol.net/articles/unofficial-bash-strict-mode/)
- [sorbet/sorbet Issue #565](https://github.com/sorbet/sorbet/issues/565)