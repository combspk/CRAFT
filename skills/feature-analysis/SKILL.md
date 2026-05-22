---
name: feature-analysis
description: Analyze existing code for bugs, UX problems, missing features, and performance issues; produce a prioritized improvement backlog
license: MIT
compatibility: opencode
---

## What I do

Perform a structured analysis of existing code to surface bugs, UX gaps, missing features, and performance or maintainability problems. Produce a prioritized backlog so the highest-value improvements are implemented first.

## When to use me

Load this skill at step 4 of the agent workflow (Improvement analysis), after all accessibility issues from step 3 have been resolved. Reload as needed when scope changes.

## Analysis procedure

Evaluate the code across each dimension below. For every finding, record:
- **Category** (Bug / UX / Feature / Performance / Maintainability / Security)
- **Priority** (Critical / High / Medium / Low)
- **Description** — what is wrong or missing, and why it matters to the user
- **Suggested implementation** — concrete approach, not vague advice

---

### 1. Bugs & Correctness
- Off-by-one errors, null/undefined dereferences, unhandled promise rejections.
- Edge cases not covered by the current logic (empty input, max values, concurrent operations).
- State mutation where immutability is expected.
- Race conditions or missing loading/error states.
- Incorrect assumptions about data shape (missing validation).

### 2. UX & Usability
- Actions with no feedback (missing loading spinners, success/error messages).
- Confusing or ambiguous labels, placeholders, and button text.
- Missing empty states (what does the UI show when a list is empty?).
- Destructive actions with no confirmation dialog.
- Forms that reset unexpectedly or lose user input on error.
- Poor mobile/responsive behavior.
- Overly long or multi-step flows that could be simplified.

### 3. Missing Features
- Functionality implied by existing UI but not yet implemented.
- Filtering, sorting, pagination on data-heavy views.
- Export or download options where users would expect them.
- User preferences or settings that would meaningfully reduce friction.
- Search capability on large datasets.
- Undo/redo for destructive or complex operations.

### 4. Performance
- Unnecessary re-renders or recomputations (React, Shiny, etc.).
- Unthrottled event handlers (scroll, resize, input).
- Large synchronous operations blocking the UI thread.
- Missing memoization or caching for expensive computations.
- Overfetching: loading more data than the view needs.
- Images or assets not optimized or lazily loaded.

### 5. Maintainability & Code Quality
- Magic numbers or strings that should be named constants.
- Duplicated logic that should be extracted into a shared function.
- Functions or components doing too many things (violates single responsibility).
- Missing or misleading comments on non-obvious logic.
- Inconsistent naming conventions within the same file.
- Dead code (unreachable branches, unused variables/imports).

### 6. Security
- User input rendered without sanitization (XSS risk).
- Sensitive data (tokens, PII) logged or exposed in error messages.
- Missing input length limits or type validation.
- CSRF exposure in forms making state-changing requests.

---

## Output format

Produce a prioritized backlog table, then a recommended implementation order.

**Backlog**:
| Priority | Category | Description | Suggested implementation |
|----------|----------|-------------|--------------------------|

**Recommended implementation order**
List items from the table in the order they should be tackled, with brief rationale for ordering (e.g., "fix the null-dereference bug first — it causes a crash that blocks all other testing").

Focus on the top 5–10 items for the next implementation pass. Do not overwhelm with low-priority items if higher-priority work remains.
