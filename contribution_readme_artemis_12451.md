# Contribution 2: `Programming Exercise`: Bonus Points: 1e-30 is considered a valid value

**Contribution Number:** 2
**Student:** Linh Nguyen
**Issue:** https://github.com/ls1intum/Artemis/issues/12451
**Status:** Phase III Complete

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

- Cloned my fork and created a working branch (`fix-issue-12451`) off `develop` (Artemis's default branch is `develop`, not `main` — confirmed on the GitHub repo before branching).
- Node/npm version mismatch: `package.json` (`engines.node`) and `.nvmrc` both pin `24.18.0`, but the Node available on my machine by default was `v20.19.0`. I have multiple Node versions installed via `nvm` (16, 20, 22, 23) but not 24.18.0 specifically — this is exactly the "Node/npm version mismatch" case called out in the assignment's troubleshooting table, and would need `nvm install 24.18.0 && nvm use 24.18.0` (plus `corepack enable` to get the pinned `pnpm@11.9.0`) to fully match the required toolchain.
- Given that mismatch, and that this specific bug (issue #12451) is a pure client-side form-validation gap with no server or database involvement, I decided it wasn't worth burning setup time forcing a mismatched Angular/TypeScript toolchain to build the *entire* app (which also needs Docker + PostgreSQL/MySQL and a full login flow) just to click into one `<input>` field. Instead, I verified the bug directly against the real production markup (see "Steps to Reproduce" below) — this is faster, fully deterministic, and isolates the bug to exactly the layer it lives in (native HTML `<input type="number">` parsing), rather than depending on the whole stack behaving correctly.
- This also matches a hint from the CI config: `.github/workflows` and `CLAUDE.md` both confirm the project's real setup commands (`pnpm start`, `./gradlew bootRun`) match what's documented, so the README wasn't stale — the friction was purely local (Node version), not a documentation gap in the project itself.

### Steps to Reproduce

1. Open the Artemis client (`pnpm start`) and log in as an instructor, or open an already-running instance.
2. Go to a course → **Exercises** → create a new **Programming Exercise** (or edit an existing one) and advance to the **Grading** step of the editor. This step is rendered by `programming-exercise-grading.component.html`.
3. Locate the **Points** field (`id="field_points"`, lines 21–31 of that template) and the **Bonus Points** field (`id="field_bonusPoints"`, lines 42–52). Both are plain `<input type="number">` elements constrained only by `min`/`max` HTML attributes — no `pattern` and no custom validator.
4. Click into **Bonus Points** and type `1e-30`, then click away (so Angular marks the control `touched`).
5. **Observe:** no red validation message (`artemisApp.exercise.bonusPointsError`) appears, and the field is treated as valid — the browser's native number parser (`input.valueAsNumber`) accepts `1e-30` as an ordinary, in-range (`> 0`) floating-point number, and Angular's `ngModel`/`MinValidator`/`MaxValidator` only check that already-parsed value against the `min`/`max` bounds, so nothing rejects the scientific-notation *format*.
6. Repeat with the **Points** field using `1e2` (`= 100`, in-range for `min=1 max=9999`): it is silently accepted the same way — confirming this is not a "Points works, Bonus Points doesn't" discrepancy; both fields share the identical gap.
7. Repeat steps 4–6 a second time (fresh page load) to confirm the behavior is deterministic, not a fluke.

I verified this exact sequence by reconstructing the real markup from `programming-exercise-grading.component.html` (identical `type`, `min`, `max` attributes) in an isolated page and driving it with a browser, since the underlying bug is 100% contained in that native `<input>` element and does not depend on any other Artemis-specific wiring — reproducing it this way is equivalent to reproducing it in the running app, but avoids needing the full Docker/Postgres/Spring Boot stack for a client-only bug.

### Reproduction Evidence

- **Branch in my fork:** https://github.com/LinhNguyen2901/Artemis/tree/fix-issue-12451
- **Screenshots/logs:** Raw log file: [`evidence/artemis-12451/repro-validity-log.txt`](evidence/artemis-12451/repro-validity-log.txt) — captured via a browser inspecting `input.validity` on both fields after entering the values above (same validity API Angular's directives read from), run twice (initial load + fresh reload) to confirm determinism. Summary:
  ```
  points:      raw value="1e2"    parsed valueAsNumber=100     validity.valid=true  (rangeUnderflow=false, rangeOverflow=false, badInput=false)
  bonusPoints: raw value="1e-30"  parsed valueAsNumber=1e-30    validity.valid=true  (rangeUnderflow=false, rangeOverflow=false, badInput=false)
  ```
- **My findings:** The bug isn't specific to Bonus Points — Points has the exact same hole, since both fields are backed by nothing more than `type="number"` + `min`/`max`. `git blame` on the template shows the `type="number"` input was introduced on 2024-10-29 (PR #9283, "Add simple mode to create and edit view"), with `max="9999"` added later (2025-07-29) and the conditional `min` refined again on 2026-07-01 (PR #12886, "Fix mandatory points for non-graded programming exercises") — none of those follow-up changes touched number-*format* validation, so the gap has existed, unnoticed, for about 20 months across several unrelated edits to the same input.

---

## Solution Approach

### Analysis

The root cause is that the **Points** and **Bonus Points** inputs — across every exercise type that has them (programming, text, modeling, file-upload — 4 near-identical templates, no shared component) — are plain HTML `<input type="number">` elements constrained only by `min`/`max` attributes. There is no `pattern` attribute and no custom Angular validator restricting the accepted *character set/format*, and no dedicated regex constant for this exists in `input.constants.ts` (which does have one for e.g. `MAX_PENALTY_PATTERN`, but nothing for points). Per the HTML spec, a browser's native number-input parser (`input.valueAsNumber`) treats scientific-notation strings like `1e-30` or `1e2` as syntactically valid floating-point numbers. Angular's `MinValidator`/`MaxValidator` directives (driven by those `min`/`max` attributes) only compare against that *already-parsed* numeric value — so `1e-30` (≈0, but `> 0` so satisfies `min=0`) and `1e2` (`=100`, within `1–9999`) both pass their bounds checks and are reported as fully valid, with no way for the validator to ever see the raw, malformed string.

This is **not** a "Points is fine, Bonus Points is broken" discrepancy as the issue title might suggest — both fields share the identical validation gap; the issue reporter likely picked Bonus Points because an absurdly tiny value like `1e-30` there is a more obviously nonsensical case (effectively free/meaningless bonus points that display as `0` without truly being exactly `0`).

The same hole exists server-side: `BaseExercise.maxPoints`/`bonusPoints` (`src/main/java/de/tum/cit/aet/artemis/exercise/domain/BaseExercise.java`) are plain `Double` fields with no Bean Validation annotations, and the corresponding `UpdateProgrammingExerciseDTO` record declares `Double maxPoints, Double bonusPoints` the same way — so a malformed value would also pass straight through the REST API if submitted directly (bypassing the client entirely).

### Proposed Solution

Reject scientific-notation (and other non-plain-decimal) input on the Points/Bonus Points fields, consistently across all four exercise-type forms, plus add matching server-side validation as defense-in-depth (client-side validation alone can be bypassed via direct API calls). Two viable directions, not mutually exclusive:

- **Client:** Migrate the Points/Bonus Points inputs to PrimeNG's `p-inputnumber` — already used for the analogous quiz-question points field (`multiple-choice-question-edit.component.html`), whose locale-aware parser doesn't accept exponent notation typed by a user. This also aligns with the project's explicit Bootstrap/raw-HTML → PrimeNG migration guideline in `CLAUDE.md`, rather than adding a one-off regex workaround to a component type that's being phased out anyway.
- **Server:** Add explicit validation on the exercise DTOs/entity (e.g. rejecting non-finite or scientifically-notated-but-absurdly-small values, or constraining to a sane decimal precision) so the same payload sent directly to the REST API is rejected with a 400, independent of whatever the client does.

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** The Points and Bonus Points number inputs on every exercise-type edit form silently accept scientific-notation values (e.g. `1e-30`, `1e2`) as valid, because the native `<input type="number">` element's `min`/`max` validation operates on the already-parsed numeric value, not the input's format — there's no check anywhere (client or server) that rejects the exponent syntax itself.

**Match:** `multiple-choice-question-edit.component.html` (quiz question points) already uses PrimeNG's `p-inputnumber` for a conceptually identical "points" value instead of a raw `<input type="number">` — its own text-based parser/formatter doesn't accept `e`-notation keystrokes the way the native number input does. This is a real, working reference implementation in the same codebase for the same underlying problem (a "points" value that should be a plain, bounded decimal), not just a superficially similar pattern.

**Plan:**
1. Extract a shared points/bonus-points input (component or, at minimum, a shared `ValidatorFn` + regex constant in `input.constants.ts`) instead of patching the same fix into 4 separately-duplicated templates (`programming-exercise-grading.component.html`, `text-exercise-update.component.html`, `modeling-exercise-update.component.html`, `file-upload-exercise-update.component.html`).
2. Replace (or wrap) the raw `<input type="number">` for `points`/`bonusPoints` in each of those 4 templates with the shared component/validator, following the `p-inputnumber` precedent above where feasible.
3. Add server-side validation to `BaseExercise`/the per-exercise-type Update DTOs (e.g. `UpdateProgrammingExerciseDTO`) so a directly-submitted API payload with a scientific-notation or otherwise malformed points value is rejected with a 400, matching the client-side behavior.
4. Add/update translations for the existing `pointsError`/`bonusPointsError` message keys if the new validator introduces a distinct error condition.

**Implement:** [[https://github.com/LinhNguyen2901/Artemis/tree/fix-issue-12451](https://github.com/LinhNguyen2901/Artemis/tree/fix-issue-12451)]

**Review:** Before opening a PR: run `pnpm run lint` and `pnpm run prettier:check` on the touched client files, `./gradlew checkstyleMain -x webapp` and `spotlessCheck` on the touched server files; confirm no new Bootstrap classes/components are introduced (`CLAUDE.md` forbids this — PrimeNG only); confirm any new constant follows the existing `input.constants.ts` naming/pattern convention; re-read `CONTRIBUTING.md` for this repo's commit-message and PR-description conventions before drafting the PR, since I haven't opened a real PR against an open-source project before.

**Evaluate:** Client: Vitest specs on each of the 4 updated exercise-update components asserting that `field_bonusPoints`/`field_points` are reported invalid (or reject the keystroke entirely, depending on the chosen approach) for inputs like `1e-30`, `1e2`, `1E5`, and remain valid for plain decimals (`0`, `1.5`, `100`). Server: a JUnit integration test (e.g. in the relevant `*IntegrationTest` class per exercise type) that PUTs/POSTs an exercise payload with `bonusPoints: 1e-30` and asserts a `400 Bad Request`. Manual: run `pnpm start` / `./gradlew bootRun`, repeat the exact reproduction steps above, and confirm the red validation alert now appears and the wizard's Save/Next action is blocked.

---

## Testing Strategy

### Unit Tests

- [x] Client: `CustomScientificNotationValidatorDirective` accepts plain decimal values (e.g. `1.5`)
- [x] Client: rejects negative-exponent scientific notation (`1e-30`, the exact issue #12451 case)
- [x] Client: rejects positive-exponent scientific notation (`1e2`)
- [x] Client: accepts an empty value (doesn't fight the `required` validator)
- [x] Server: `Exercise.validateScoreSettings()` rejects `maxPoints = 1e-30`
- [x] Server: rejects `bonusPoints = 1e-30`
- [x] Server: accepts normal-precision points (`100.5`, `0.25`) as a regression guard

### Integration Tests

- [x] Ran the existing Vitest suites for all 4 touched components (`programming-exercise-grading`, `text-exercise-update`, `modeling-exercise-update`, `file-upload-exercise-update`) — 92 pre-existing tests, all still passing, confirming the new directive doesn't break existing form behavior
- [x] Ran `ExerciseTest.java` via `./gradlew test --tests ExerciseTest -x webapp` (Testcontainers/PostgreSQL) — 16/16 tests passing (13 pre-existing + 3 new)

### Manual Testing

The local toolchain (Node, Docker, JDK) only came together gradually during this session, so I verified the core logic manually before the full automated suites were runnable:
- Confirmed the directive's exact logic (reading the raw input value and rejecting scientific notation) against real browser DOM behavior, using an isolated HTML page reconstructing the actual production `<input>` markup — `1e-30` and `1e2` were both correctly flagged.
- Confirmed the server-side `BigDecimal` precision check with a standalone Java snippet before the Gradle toolchain was working: all legitimate points values (`0`, `1.5`, `100.5`, `9999`) passed, and all scientific-notation/absurd-precision values (`1e-30`, `1e-10`, `123.456789`) correctly failed.
- Once the toolchain was fully working, ran the real Vitest and Gradle test suites (see Integration Tests above) for authoritative, automated confirmation of the same behavior.

---

## Implementation Notes

### Week 7 Progress (last week — reproduction)

**What I did:**
- Reproduced the issue: confirmed both the `bonusPoints` and `points` fields silently accept scientific-notation input (e.g. `1e-30`, `1e2`) as valid, using an isolated browser harness that reconstructed the real production `<input type="number">` markup — verified twice for determinism (see Reproduction Process above).
- Used `git blame` to confirm the underlying markup has existed unnoticed since 2024-10-29 (PR #9283), across several unrelated follow-up edits, and confirmed via code inspection that this isn't a "Points works, Bonus Points doesn't" discrepancy — both fields share the identical gap.
- Wrote the solution plan (UMPIRE) identifying the root cause (validators only ever see the already-parsed value, never the raw typed string) and a "Match" reference in the existing codebase (`p-inputnumber`, used for quiz question points) before writing any code.

### Week 8 Progress (this week — implementation)

**What I did:**
- Added `CustomScientificNotationValidatorDirective` (`custom-scientific-notation-validator.directive.ts`) — a shared Angular validator that reads the raw native input value and rejects scientific-notation strings (e.g. `1e-30`, `1e2`), since native `<input type="number">` only ever exposes the already-parsed number to `min`/`max` validators.
- Wired the new directive (`noScientificNotation` attribute) onto the `points`/`bonusPoints` fields in all 4 exercise-type forms: `programming-exercise-grading.component.html`, `text-exercise-update.component.html`, `modeling-exercise-update.component.html`, `file-upload-exercise-update.component.html`.
- Added a server-side precision check in `Exercise.validateScoreSettings()` (`Exercise.java`) rejecting `maxPoints`/`bonusPoints` with more than 4 decimal places, as defense-in-depth since the client-side check can be bypassed via direct API calls.
- Added 4 Vitest tests for the new directive and 3 JUnit tests in `ExerciseTest.java` (see Testing Strategy above), then got the local toolchain (Node `24.18.0`, Docker, JDK 25) working so both suites, plus the existing 92 tests across the 4 touched components, could actually run — all passing, no regressions.
- Committed the work as two focused commits and pushed both to my fork's `fix-issue-12451` branch.

**Challenges faced:**
- The core challenge was realizing that Angular's validator mechanism (`NG_VALIDATORS`, and any custom `ValidatorFn`) only ever sees the FormControl's *already-parsed* value, never the raw string the user typed — by the time a validator runs, `"1e-30"` and its decimal expansion are indistinguishable, both just the JS number `1e-30`. That meant a "normal" custom validator couldn't work at all; the fix had to read the native `<input>` element's raw `.value` string directly inside the validator (via `ElementRef`), which is the only place the original typed format still exists.
- Writing the automated regression test for this surfaced a subtlety worth noting briefly: simulating a real keystroke (raw DOM value + dispatched `input` event) only works if it happens *after* Angular's first `detectChanges()` — otherwise `NgModel` hasn't wired up its `ControlValueAccessor` yet, and the simulated input silently goes nowhere. Took some iteration to isolate.

**Commits this week:**
- `ec9a7557bf`: Reject scientific notation in points/bonus points fields
- `4752b42617`: Reject points with too many decimal places on the server too

### Code Changes

- **Files modified:**
  - `src/main/webapp/app/foundation/validators/custom-scientific-notation-validator.directive.ts` (new)
  - `src/main/webapp/app/foundation/validators/custom-scientific-notation-validator.directive.spec.ts` (new)
  - `src/main/webapp/app/programming/manage/update/update-components/grading/programming-exercise-grading.component.{html,ts}`
  - `src/main/webapp/app/text/manage/text-exercise/update/text-exercise-update.component.{html,ts}`
  - `src/main/webapp/app/modeling/manage/update/modeling-exercise-update.component.{html,ts}`
  - `src/main/webapp/app/fileupload/manage/update/file-upload-exercise-update.component.{html,ts}`
  - `src/main/java/de/tum/cit/aet/artemis/exercise/domain/Exercise.java`
  - `src/test/java/de/tum/cit/aet/artemis/exercise/ExerciseTest.java`
- **Key commits:**
  - [`ec9a7557bf`](https://github.com/LinhNguyen2901/Artemis/commit/ec9a7557bf9b6fddd1dcdf41b246b4c06bc44614) — client-side directive + wiring into the 4 exercise forms
  - [`4752b42617`](https://github.com/LinhNguyen2901/Artemis/commit/4752b42617c911bf60e7dfb62d70eafb1492151f) — server-side decimal-precision check
- **Approach decisions:** Chose a shared validator directive (reading the raw DOM value) over migrating to PrimeNG's `p-inputnumber` — the "Match" option identified in the Phase II plan. The directive is a smaller, more targeted change that fixes the exact reported defect without a UI/styling migration risk across 4 separate forms; the PrimeNG migration remains a valid follow-up but felt like scope creep for this specific bug fix. For the server-side fix, chose a decimal-precision check (max 4 places) rather than trying to reject scientific-notation *syntax* directly, since by the time a JSON payload is deserialized into a `Double`, the original string format is already gone — precision is the only signal that survives to the server.

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