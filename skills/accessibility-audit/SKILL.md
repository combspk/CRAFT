---
name: accessibility-audit
description: Audit code for Section 508 and WCAG 2.0 Level A/AA compliance; produce a prioritized list of violations and fixes
license: MIT
compatibility: opencode
metadata:
  standards: Section 508 (36 C.F.R. Part 1194), WCAG 2.0 Level A and AA
  reference: https://www.section508.gov/develop/guide-accessible-web-design-development/
---

## What I do

Systematically audit HTML, CSS, and JavaScript/framework code against every Section 508 / WCAG 2.0 Level A and AA requirement, then produce a prioritized list of violations with exact fix instructions.

## When to use me

Load this skill at step 2 of the agent workflow (Accessibility audit) and again after every implementation pass (step 6). Use whenever code contains UI, markup, or interactive components.

## Audit procedure

Work through every category below. For each violation, record:
- **WCAG criterion** (e.g., `1.4.3`)
- **Location** (file + line/selector)
- **Issue** (what is wrong)
- **Fix** (exact change required)

### 1. Images — WCAG 1.1.1
- Every `<img>` has an `alt` attribute.
- Meaningful images: `alt` describes the content or function. Ask "what text would replace this image?"
- Decorative images: `alt=""` (empty string, not omitted).
- No images of text; use styled real text.
- CSS background images that convey meaning need an on-page text equivalent.

### 2. Color & Contrast — WCAG 1.4.1, 1.4.3
- Normal text: contrast ratio ≥ 4.5:1.
- Large text (≥ 18pt / ≥ 14pt bold): contrast ratio ≥ 3:1.
- Test with [WebAIM Contrast Checker](https://webaim.org/resources/contrastchecker/).
- Color is never the sole conveyor of meaning — always pair with text, icon, or pattern.

### 3. Keyboard & Focus — WCAG 2.1.1, 2.1.2, 2.4.3, 2.4.7, 3.2.1
- All interactive elements reachable and operable by keyboard alone (Tab, Enter, Space, arrow keys as appropriate).
- No keyboard trap: user can always Tab away from any widget.
- `tabindex` usage: `tabindex="0"` to add to tab order; `tabindex="-1"` to allow programmatic focus only. Never use large positive values.
- Focus order matches visual/logical reading order (left-to-right, top-to-bottom).
- Visible focus indicator must exist — never `outline: none` or `outline: 0` without a custom visible replacement.
- Focus does not trigger a context change (no auto-navigation, no auto-submit on focus).
- After a modal closes or an element is deleted, focus is explicitly moved to a logical target.

### 4. Forms — WCAG 1.3.1, 3.2.2, 3.3.1, 3.3.2, 3.3.3, 3.3.4
- Every `<input>`, `<select>`, `<textarea>` has an associated `<label for="id">` or `aria-label`/`aria-labelledby`.
- Related controls grouped in `<fieldset>` with `<legend>` (radio groups, checkboxes, address fields).
- Required fields marked with `aria-required="true"`.
- Input errors: identified by text (not color alone), describe the problem, suggest a fix if known.
- `role="alert"` on error containers so screen readers announce them immediately.
- Best practice: move focus to the first error or error summary on submission failure.
- Legal/financial/deletion forms: must be reversible, checkable, or include a review step (choose one).
- Context changes (new window, page nav) not auto-triggered on input change — require an explicit button.

### 5. Headings & Structure — WCAG 1.3.1, 1.3.2, 2.4.6
- One `<h1>` per page/view.
- Heading levels are sequential — never skip (e.g., `h2` → `h4`).
- Headings used only for actual headings, not styling.
- Reading order in DOM matches visual order (avoid `position: absolute` reordering).
- Avoid CSS `::before`/`::after` pseudo-elements for meaningful content.

### 6. Page & Navigation — WCAG 2.4.1, 2.4.2, 2.4.4, 2.4.5, 3.2.3, 3.2.4
- `<title>` is descriptive and unique per page/view; updates on SPA navigation.
- Skip-navigation link (or landmark regions) provided to bypass repeated nav blocks.
- Link and button text is descriptive standalone — no "click here", "read more", "here".
- Navigation repeated across pages is in consistent order.
- Multiple ways to reach a page (nav menu + search, or sitemap, etc.).

### 7. Dynamic Content & ARIA — WCAG 4.1.2
- Custom widgets expose name, role, state, and value (`aria-*` attributes or native semantics).
- Status messages use `aria-live="polite"`; urgent alerts use `role="alert"` (`aria-live="assertive"`).
- Modal dialogs: `role="dialog"`, `aria-modal="true"`, `aria-labelledby` pointing to dialog title, focus trapped inside while open, focus restored on close.
- Expanded/collapsed state: `aria-expanded="true|false"`.
- Selected/checked state: `aria-selected`, `aria-checked` as appropriate.
- Never use ARIA to override semantics when a native HTML element exists.

### 8. Language — WCAG 3.1.1, 3.1.2
- `<html lang="en">` (or correct BCP 47 tag) always present.
- Inline passages in another language: `lang` attribute on the containing element.

### 9. Tables — WCAG 1.3.1
- Data tables: `<th scope="col">` and/or `<th scope="row">` on all header cells.
- Complex tables (multi-level headers): use `id`/`headers` associations.
- Layout tables: `role="presentation"`, no `<th>`, no `scope`, no `summary`.

### 10. Media — WCAG 1.2.1–1.2.5, 1.4.2, 2.2.2, Section 508 §503.4
- Prerecorded audio-only: text transcript linked adjacent to player.
- Prerecorded video-only: text transcript or audio track.
- Synchronized video: closed captions + audio descriptions.
- Live audio: real-time captions.
- Auto-playing audio > 3 s: pause/stop/volume control provided.
- Moving/blinking/scrolling content lasting > 5 s: pause/stop/hide control.
- Media player exposes CC and AD controls at the same menu level as volume/program controls (§503.4).

### 11. Flashing — WCAG 2.3.1
- No content flashes more than 3 times per second.

### 12. Timed Events — WCAG 2.2.1
- Time limits can be turned off, adjusted (≥ 10× default), or extended (≥ 20 s warning, ≥ 10 extensions) — unless real-time or > 20 hours.

### 13. Frames & iFrames — WCAG 2.4.1, 4.1.2
- Every `<iframe>` has a descriptive `title` attribute.
- `<frame>` elements are obsolete; replace with native elements.

### 14. Parsing & Code Quality — WCAG 4.1.1
- All tags properly opened and closed.
- No duplicate `id` values on the same page.
- No duplicate attributes on any element.
- Validate HTML; structural errors break assistive technology parsing.

### 15. Text Resize — WCAG 1.4.4
- Page remains functional and readable at 200% browser zoom.
- Use relative units (`em`, `rem`, `%`) for font sizes, padding, and layout widths.

### 16. Sensory Independence — WCAG 1.3.3
- Instructions never rely solely on shape, size, color, position, or sound (e.g., avoid "click the green button on the right").

## Output format

After completing the audit, produce two sections:

**Violations** (table or numbered list):
| # | Criterion | File/Selector | Issue | Fix |

**Summary**
- Total violations found
- Critical (blocks assistive technology) vs. non-critical
- Recommended fix order (highest impact first)
