# Contribution [#]: [Issue Title]

**Contribution Number:** 1 
**Student:** Linh Nguyen
**Issue:** [\[GitHub issue link\]](https://github.com/graphql-hive/console/issues/2730)  
**Status:** Phase II

---

## Why I Chose This Issue

This issue asks for the Hive UI to support custom HTTP headers for each target during external schema composition, so users can apply target-specific transform behavior and authentication when composing remote schemas. It matters because external schema composition often depends on per-target metadata, and without per-target headers Hive cannot correctly compose or transform some external sources. 
I chose it because it's a concrete UI + integration enhancement with a clear developer impact, it matches the console's frontend/settings and schema composition stack I want to work on, and it's a great way to improve Hive's composability while gaining experience with the exact tech area I'm interested in.


---

## Understanding the Issue

### Problem Description

[In your own words, what's broken or missing?]

### Expected Behavior

[What should happen?]

### Current Behavior

[What actually happens?]

### Affected Components

[Which parts of the codebase are involved?]

---

## Reproduction Process

### Environment Setup

[Notes on setting up your local development environment - challenges you faced, how you solved them]

### Steps to Reproduce

1. Clone your fork and start the local dev environment:
   ```
   git clone https://github.com/<your-username>/console.git
   cd console
   pnpm install
   docker compose up -d
   pnpm dev
   ```
2. Open the Hive UI at `http://localhost:3000` and sign in.
3. Create an Organization, then create a Project with type **Federation**.
4. Inside the project, create a Target (e.g., "production").
5. Navigate to **Project Settings** → scroll to the **External Schema Composition** section.
6. Observe that only two fields exist: **Endpoint URL** and **Secret**. There is no field for custom HTTP headers.
7. Now navigate to **Target Settings** for the target you created.
8. Observe that there is no External Schema Composition section at all on the target settings page — no way to configure per-target custom headers.
9. Attempt to compose schemas for the target; note that no custom headers can be injected into the outbound composition request.

**Observed result:** There is no UI or API to provide custom HTTP headers per target for the external schema composer. All targets share the same project-level endpoint and secret with no way to customize request headers per target.

### Reproduction Evidence

- **Branch:** [https://github.com/nguyenthuylinh280105/console/tree/fix-issue-2730](https://github.com/nguyenthuylinh280105/console/tree/fix-issue-2730)
- **Screenshots/logs:** N/A — feature is entirely absent (no UI element to screenshot)
- **My findings:** Confirmed by reading `packages/web/app/src/pages/target-settings.tsx` (no composition headers section), `packages/services/api/src/modules/target/module.graphql.ts` (no custom-headers mutation), and `packages/services/schema/src/lib/compose.ts` where the outbound request headers are hardcoded to `Accept`, `Content-Type`, and `x-hive-signature-256` only.

---

## Solution Approach

### Analysis

[Your analysis of the root cause - what's causing the issue?]

### Proposed Solution

[High-level description of your fix approach]

### Implementation Plan

Using UMPIRE framework (adapted):

**Understand:** Composition settings (endpoint + secret) are project-level. There is no way to attach custom HTTP headers to the outbound composition request per target.

**Match:** `graphql_endpoint_url` follows the same per-target pattern — nullable column on `targets`, GraphQL mutation in `target/module.graphql.ts`, input field in `target-settings.tsx`. Custom headers will use this same pattern.

**Plan:**
1. Add DB migration: `external_composition_custom_headers JSONB NULL` on `targets`.
2. Update storage types + `getTarget` query; add `updateTargetExternalCompositionCustomHeaders()` method.
3. Add `updateTargetExternalCompositionCustomHeaders` mutation to `target/module.graphql.ts`.
4. Extend `ExternalComposition` type in `schema/src/types.ts` with `customHeaders`; thread it through `composition-orchestrator.ts` into `composeExternalFederation` in `compose.ts`, merging before Hive's own headers so the signature always wins.
5. Add a key-value headers input to `target-settings.tsx`, visible only when external composition is enabled.
6. Add an integration test to `external-composition.spec.ts` verifying custom headers reach the mock composer.

**Implement:** [Link to your branch/commits as you work]

**Review:** `pnpm lint` + `pnpm typecheck`; existing external composition tests pass; conventional commits.

**Evaluate:** Unit test header merging; integration test end-to-end; manual UI test.

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