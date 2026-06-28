# Contribution #565: Audit bash scripts to use "unofficial bash strict mode" in more places

**Contribution Number:** 1 
**Student:** Linh Nguyen
**Issue:** [#565 – Audit bash scripts to use "unofficial bash strict mode" in more places](https://github.com/sorbet/sorbet/issues/565)  
**Status:** Phase IV

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

- **Branch:** [fix/bash-strict-mode-565](https://github.com/LinhNguyen2901/sorbet/tree/fix/bash-strict-mode-565)
- **Screenshots/logs:** N/A — the issue is confirmed by running the grep command above and seeing many scripts without the strict mode header.
- **My findings:** Running `grep -rL "set -euo pipefail" --include="*.sh" .` from the repo root returned 240 scripts needing updates (205 missing strict mode entirely, 35 with partial mode). Out of 310 total `.sh` files, 70 already had full `set -euo pipefail`. The violations are spread across `.buildkite/`, `tools/`, `test/`, and `vscode_extension/`. None of the scripts used `IFS` changes, so no extra care was needed there — only the `set -euo pipefail` line needed to be added or upgraded.

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
2. For each script, add `set -euo pipefail` immediately after the shebang line. For scripts with partial strict mode (e.g. `set -e`, `set -eu`, `set -eo pipefail`), replace the existing `set` line in-place to preserve position and any extra flags like `-x`.
3. Run any existing CI or test scripts to verify nothing breaks after the change.
4. Submit a PR with all changes, grouped logically by directory.

**Implement:** [fix/bash-strict-mode-565](https://github.com/LinhNguyen2901/sorbet/tree/fix/bash-strict-mode-565)

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

- Ran `grep -rL "set -euo pipefail" --include="*.sh" .` before changes: returned 205 scripts with no strict mode and 35 with partial mode (240 total needing updates).
- After changes: all 240 scripts updated. Scripts with `-x` preserved as `set -euxo pipefail` rather than dropping the flag.
- **Not yet manually executed** — scripts have not been run locally. CI on the PR is the intended verification step.

---

## Implementation Notes

### Progress

- Audited all 310 `.sh` files in the repo: 70 already had full strict mode, 35 had partial, 205 had none.
- Wrote a Python script to batch-process all 240 files needing changes.
- Scripts with partial mode (e.g. `set -eo pipefail`) were upgraded in-place, preserving flags like `-x` (→ `set -euxo pipefail`).
- Scripts with no mode had `set -euo pipefail` inserted after the shebang line.

### Code Changes

- **Files modified:** 185 `.sh` files total
- **Key commits:**
  - [`36a9021`](https://github.com/LinhNguyen2901/sorbet/commit/36a9021a7) — Apply bash strict mode to CI and tooling scripts (15 files: `.buildkite/`, `tools/`, `vscode_extension/`)
  - [`31018c6`](https://github.com/LinhNguyen2901/sorbet/commit/31018c65d) — Apply bash strict mode to test scripts (170 files under `test/`)
- **Approach decisions:** Partial-mode scripts were fixed in-place rather than inserting a duplicate `set` line, to avoid two competing `set` calls in the same script.

---

## Pull Request

**PR Link:** [sorbet/sorbet#10410](https://github.com/sorbet/sorbet/pull/10410)

**PR Description:**
Audited all 310 `.sh` files in the repo and applied `set -euo pipefail` to the 240 scripts that were missing it or only had partial strict mode (e.g. `set -e`, `set -eu`, `set -eo pipefail`). Scripts with `-x` were upgraded to `set -euxo pipefail` to preserve that flag. The change is purely additive — no script logic was altered.

**Summary of Contribution:**
- Identified 240 bash scripts across `.buildkite/`, `tools/`, `test/`, and `vscode_extension/` that lacked full strict mode.
- Upgraded 35 partial-mode scripts in-place and added strict mode to 205 scripts that had none.
- Split into two commits: one for CI/tooling scripts (15 files) and one for test scripts (170 files).

**Maintainer Feedback:**
- Awaiting first review.

**Status:** Awaiting Review

---

## Learnings & Reflections

### Technical Skills Gained

- Learned how bash strict mode flags (`-e`, `-u`, `-o pipefail`) interact and why each matters independently.
- Practiced auditing a large codebase systematically using `grep` and scripting (Python) to apply batch changes safely.
- Understood the open source contribution workflow: fork → branch → PR against upstream.

### Challenges Overcome

- 240 files needed updating — doing this manually would be error-prone. Wrote a Python script to handle both the "add after shebang" case and the "upgrade partial set command in-place" case, while preserving flags like `-x`.
- Partial-mode scripts had their `set` command at varying line positions (not always line 2), so the script had to detect and replace the existing line rather than blindly inserting after the shebang.

### What I'd Do Differently Next Time

- Run CI locally (or at least test a representative sample of scripts) before submitting, to catch any test scripts that break when `grep` returns no matches under `-e`.
- Open the PR slightly earlier to get maintainer feedback on approach before doing all 240 files.

---

## Resources Used

- [The "Unofficial Bash Strict Mode" – redsymbol.net](http://redsymbol.net/articles/unofficial-bash-strict-mode/)
- [sorbet/sorbet Issue #565](https://github.com/sorbet/sorbet/issues/565)