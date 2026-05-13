---
name: vue-audit
description: Audits a single .vue file for performance issues and code-reuse opportunities, producing a prioritized read-only report. Use when the user asks to audit, review, analyze, or optimize a Vue component, or says things like "what could be improved in this view", "any reuse opportunities in X.vue", "is this component performant". Requires a file path as an argument — error out if none is given. Does not modify files; the user applies fixes manually or via vue-expert.
---

# Vue Component Audit

Audit a single `.vue` file against this project's conventions and produce a prioritized markdown report. **Read-only — do not edit the file.**

## Invocation

The user invokes this skill with a file path:

```
/vue-audit client/src/views/Orders.vue
```

**If no path is given, stop and ask for one.** Do not pick a file yourself — the user knowing which file to audit is part of the input. A blanket "audit everything" run is out of scope for this skill (use `/optimize` for that).

**If the path doesn't end in `.vue` or doesn't exist, stop and report the problem.** Don't audit non-Vue files.

## What this skill is not

- Not a fix tool. Never `Edit` or `Write` the target file. The report is the output.
- Not a whole-codebase audit. One file at a time, by design — keeps the report grounded and the context cost predictable.
- Not a replacement for `code-reviewer`. That agent gives a generalist review; this skill is Vue-specific and uses this project's `client/CLAUDE.md` conventions as the rubric.

## Workflow

1. **Read the target file in full.** Don't sample — a v-for issue on line 240 matters as much as one on line 12.
2. **Read `client/CLAUDE.md`** if you haven't this session. It is the authoritative rubric for project conventions.
3. **Walk the checks below** and collect findings. Each finding needs a concrete line number and a quoted snippet.
4. **Cross-check templates against `client/src/locales/en.js` and `ja.js`** if the file uses `t(...)` — flag any missing keys.
5. **For reuse findings, grep the rest of `client/src/`** to confirm duplication exists elsewhere before claiming "this is duplicated." A pattern that only appears once isn't a reuse opportunity.
6. **Emit the report in the exact format below.** Don't editorialize beyond it.

## Severity rubric

Pick one per finding. Be honest — over-severing trains the user to ignore the report.

- **High** — Causes a visible bug, a measurable perf regression, or violates a hard project rule documented in `client/CLAUDE.md` (e.g., `v-for` index keys, missing `.value`, month filter on inventory).
- **Medium** — A clear improvement with a real payoff but no active bug. Method-in-template that should be computed, duplicated logic that belongs in a composable, missing loading/error state.
- **Low** — Style or polish. A long inline style block that could be a class, a magic number that could be a constant, a slightly awkward template structure. Easy to skip without consequence.

If you can't argue why a finding earns its severity in one sentence, it's probably one level too high.

## Checks

### Performance

| # | Check | Rubric |
|---|---|---|
| P1 | `v-for` keys use a unique field (not index) | `client/CLAUDE.md` — index keys cause Vue to reuse DOM nodes incorrectly on reorder/insert/delete. |
| P2 | Template calls methods for derived values (`{{ calculateTotal() }}`) instead of `computed` | Methods run on every render; computed caches until deps change. |
| P3 | `v-if` used where `v-show` would be cheaper (frequently-toggled element with no expensive children) | `v-show` toggles `display`; `v-if` mounts/unmounts. Inverse also a finding: heavy subtree using `v-show` when it's rarely visible. |
| P4 | Date operations without `isNaN(date.getTime())` validation | A bad date silently becomes `NaN` and breaks downstream `.getMonth()`/`.toLocaleDateString()`. |
| P5 | Inline object/array/function literals in template props (`:style="{...}"`, `@click="() => ..."` inside a `v-for`) | New reference each render — pushes child re-renders. Hoist to computed or methods. |
| P6 | Large list rendered without virtualization (>200 expected rows, no `v-memo`, no virtual scroller) | Only flag if the data source can realistically exceed ~200 items. Inventory and Orders qualify; Dashboard cards usually don't. |
| P7 | `watch` without `{ immediate: false }` or debounce on a high-frequency source (search input, slider) | Fires on every keystroke / tick. Suggest `watchDebounced` from `@vueuse/core` or a manual debounce. |
| P8 | Filter/transform chains rebuilt inside a method instead of a `computed` | Same as P2 but specifically for `.filter().map().reduce()` chains that depend only on reactive state. |

### Code reuse

Before flagging any of these, **grep the rest of `client/src/`** to confirm the pattern exists elsewhere.

| # | Check | Rubric |
|---|---|---|
| R1 | Loading / error / data refs declared inline when an identical block exists in ≥2 other views | Candidate for a `useAsyncData` composable. Cite the other files. |
| R2 | Filter logic (warehouse/category/status/month) inlined when `useFilters()` would cover it | Project already has `client/src/composables/useFilters.js`. Flag any view re-implementing filter state instead of consuming it. |
| R3 | Currency/date/percentage formatting written inline | Suggest a `formatCurrency` / `formatDate` helper if the same shape appears ≥2 places. Don't propose a util for a one-off. |
| R4 | Template chunk (>15 lines) duplicated near-verbatim in another view | Candidate for `client/src/components/`. Cite the other file. |
| R5 | Repeated `axios.get` patterns that bypass `client/src/api.js` | All HTTP should go through `api.js`. Direct axios usage in a view is a finding. |
| R6 | Inline `<style>` block re-declares utilities already in `App.vue` (`.card`, `.page-header`, `.table-container`, etc.) | Read `client/src/App.vue` first to confirm the rule exists globally before flagging. |

### Project conventions

These come straight from `client/CLAUDE.md` and the vue-expert agent. Hard rules.

| # | Check | Rubric |
|---|---|---|
| C1 | User-facing string is hardcoded instead of using `t('...')` | Every visible string should route through `vue-i18n`. |
| C2 | A `t('...')` key is referenced but missing from `client/src/locales/en.js` **or** `client/src/locales/ja.js` | Both locales must define every key. A key in only one is a runtime fallback waiting to happen. |
| C3 | Hardcoded `$` instead of the `currencySymbol` from the locale composable | Locale-dependent. Currency symbol must come from the i18n setup. |
| C4 | Emoji in template, `<style>`, or any user-visible string | Project rule: no emojis in UI. |
| C5 | Mixing Options API (`data()`, `methods:`) with Composition API (`setup()`) | `client/CLAUDE.md` mandates Composition API throughout. |
| C6 | Direct prop mutation (`props.x = ...`, `props.list.push(...)`) | Always emit. |
| C7 | Inventory view applies the `month` filter | Inventory has no time dimension. Documented bug class. |
| C8 | Component file >300 lines of template OR >300 lines of script | `client/CLAUDE.md`: extract a component at >100 template lines, a composable at >150 script lines. The >300 threshold here is the "definitely extract" line. |

## Report format

Output exactly this structure. No preamble, no closing pleasantries.

```markdown
# Vue Audit: <relative-path>

**Lines:** <total> · **Template:** <n> · **Script:** <n> · **Style:** <n>
**Findings:** <high-count> high · <medium-count> medium · <low-count> low

## High

### [P1] v-for uses index as key
**Location:** `client/src/views/Foo.vue:73`
**Snippet:**
\```vue
<tr v-for="(order, i) in orders" :key="i">
\```
**Why it matters:** When `orders` reorders (e.g., after a filter change), Vue reuses DOM rows by position, which can swap children of `<details>` elements and leak open/closed state across rows.
**Suggested fix:** Key by `order.id` — already unique on the API response.

### [C2] Missing locale key
... (one block per finding)

## Medium

... (same format)

## Low

... (same format)

## Reuse opportunities (cross-file)

For R1–R5 findings, list the related files explicitly:

### [R2] View re-implements filter state instead of using useFilters()
**This file:** `client/src/views/Foo.vue:42-68` declares `selectedWarehouse`, `selectedCategory` as local refs.
**Also seen in:** `client/src/views/Bar.vue:31-55`, `client/src/views/Baz.vue:22-44`.
**Suggested fix:** Replace local refs with `const { selectedWarehouse, selectedCategory } = useFilters()`. The composable already exists at `client/src/composables/useFilters.js`.

## Not flagged (but considered)

Brief 1–3 bullet list of checks that passed cleanly, *only* if they were non-trivially correct. Skip the trivial passes. This section gives the user confidence the audit was thorough; padding it defeats the point.
```

## Tone

Direct. No "great work overall!" framing. No emojis. Quote actual code from the file rather than paraphrasing — a real snippet anchors every claim.

If the file is genuinely clean, the report can be three lines plus the "Not flagged" section. Don't manufacture findings to fill the rubric.

## Suggested follow-up

End the report (after the markdown block above) with a one-liner the user can act on, e.g.:

> To apply the high-severity fixes, run: *delegate to vue-expert with these findings: [P1] line 73, [C2] missing key …*

Don't delegate yourself. The user asked for a report; the report is the deliverable.
