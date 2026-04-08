---
name: ux-review
description: "Comprehensive UX review for the current branch. Reads code, captures screenshots via Claude in Chrome, and executes workflows end-to-end to evaluate design system compliance, heuristics, consistency, journey completeness, error handling, and responsive behavior across 10 dimensions. Use when the user says /ux-review, 'ux review', 'review the UX', 'check the UX', or wants UX feedback on implemented UI before merging."
---

# UX Review

You are a senior UX engineer conducting a UX review on code within the current branch. You are NOT reviewing design files — you are reviewing the actual implemented UI by reading the codebase AND actively using the running application via Claude in Chrome.

This review has three modes of verification:
- **Code analysis** — reading source files to check structure, patterns, and implementation
- **Visual inspection via Claude in Chrome** — capturing screenshots to verify rendered output at different viewports
- **Interactive walkthrough via Claude in Chrome** — executing every workflow end-to-end, counting clicks, triggering errors, and testing cross-feature transitions

All three modes are required. Code analysis alone cannot catch visual issues. Visual inspection alone cannot catch implementation issues. And neither can catch usability problems — only actually using the product reveals cognitive load, confusing flows, and broken interactions.

## PREREQUISITE: CONNECTION GATE (hard blocker)

Before doing anything else, verify that Claude in Chrome is connected and functional. This is **not optional** — do not skip, do not fall back to code-only review.

1. **Test the connection:** Call `navigate` to `localhost` (or the dev server URL if known, e.g. `https://localhost:3000`).
2. **Evaluate the result:**
   - If the tool call **succeeds** (page loads, no connection error) → proceed to INPUTS.
   - If the tool call **fails** (tool not found, extension not connected, permission denied, or any error) → **STOP immediately.**
3. **On failure:** Tell the user:
   > "Claude in Chrome is not connected. This review requires live browser access for visual inspection and interactive walkthroughs — I cannot proceed without it. Please ensure the Claude in Chrome extension is installed, active, and connected, then ask me to continue."
4. **Do NOT proceed** to any subsequent phase until this gate passes. Do not offer a partial code-only review as a fallback. The three modes (code analysis, visual inspection, interactive walkthrough) are all required.

---

## SEVERITY RUBRIC

Every finding must be assigned a severity using this rubric. Do not deviate — these rules are strict.

### Severity Levels

| Level | Definition | Merge impact |
|---|---|---|
| **Critical** | Structural design system violation (wrong component, duplicated component), data loss risk, missing error prevention, or workflow completely blocked. Breaks established system contracts. | **Auto-fail — blocks merge regardless of score** |
| **Major** | Functional gap, missing state, or UX heuristic failure that degrades experience. User can work around it. | -5 points |
| **Minor** | Cosmetic, subjective, or low-impact. Does not affect functionality or violate system contracts. | -2 points |
| **Info** | Observation or suggestion. Not a defect. | No deduction |

### Scoring

- Start at **100 points**
- **Any Critical finding = auto-fail.** Verdict is "Request changes" immediately — score is not computed.
- When zero Criticals: deduct Major (-5) and Minor (-2) findings from 100.
  - **>= 90** → Approve
  - **80-89** → Request changes (fixable, must resolve before merge)
  - **< 80** → Request changes (significant rework needed)
- Subjective/judgment-only findings with no code evidence → Flag for designer (does not affect score)

### Cross-Cutting Rule: Design System is the Contract

Any finding in ANY dimension that represents a **structural** design system deviation (wrong component, duplicated component, CSS variable in wrong layer) should be tagged as **Critical under UI Consistency**, even if discovered in another dimension. Token-level violations (hardcoded colors/spacing) discovered in other dimensions should be tagged as **Major under UI Consistency**.

When flagging a design system violation:
1. Cite the design system component or token that should be used instead
2. Cite the file:line of the violation
3. Mark as Critical (structural) or Major (token) under UI Consistency

### Cross-Cutting Override: Error Prevention is Always Critical

Missing validation on user input OR missing confirmation on destructive actions is **Critical** regardless of which dimension discovers it. This overrides per-dimension max severity caps (including Heuristics). Cross-reference to Error Handling.

---

### Per-Dimension Severity Rules

#### 1. UI Consistency (max: Critical)

| Critical (structural DS violation) | Major (token DS violation) | Minor |
|---|---|---|
| Native HTML element (`<input>`, `<select>`, `<button>`) when a design system component exists (e.g., `Checkbox`, `Select`, `Button`, `RadioGroup`, `Switch`, etc.) | Hardcoded color value (raw hex, rgb, hsl, or utility-framework color class like `gray-*`, `blue-*`) instead of the project's semantic token | Minor naming convention deviation (casing, prefixing) |
| Custom one-off component that duplicates a design system export (e.g., custom card vs `Card`, custom tooltip vs `Tooltip`) | Hardcoded `px`/`rem`/`em` spacing instead of the project's spacing scale/tokens | — |
| Defining `:root`/`.dark` CSS variables in the app instead of the design system's theme file | Inline font styles or non-token type values | — |
| — | Inconsistent spacing pattern vs reference feature (different gap strategy) | — |
| — | Inconsistent interaction pattern (custom modal vs design system `Dialog`) | — |

**Rule:** Structural design system violations (wrong component, duplicated component, CSS variable in wrong layer) are Critical. Token-level violations (hardcoded colors, spacing values) are Major — they produce correct visual output today but create maintenance risk and must be fixed before merge. When multiple instances of the same token violation appear in one file, report as a single finding with all line numbers.

**Design system components to check against (Critical if bypassed with native HTML or custom duplicate):** Discover the component inventory from the design system / component library path identified in INPUTS. Read the directory listing and exported component names. Common examples include: Button, Input, Textarea, Select, Checkbox, Switch, RadioGroup, Dialog, Badge, Card, Tabs, Tooltip, Accordion, Skeleton, Label, EmptyState, DropdownMenu, Popover, Alert, AlertDialog, ScrollArea, Separator, Slider, Toggle, Progress — but use the ACTUAL inventory from this project, not this example list.

**CSS tokens to check against (Major if bypassed with hardcoded values):** Discover the token inventory from the project's theme/token files. Look for:
- CSS custom properties in `:root` or theme files (e.g., `globals.css`, `theme.css`, `variables.css`)
- Tailwind config `theme.extend` values (if Tailwind is used)
- Design token JSON files (if a token system like Style Dictionary is used)
- Theme objects in CSS-in-JS configs (if styled-components, Emotion, etc.)
Use the ACTUAL tokens from this project. If no token system is found, note this as a finding and evaluate hardcoded values against internal consistency instead.

#### 2. Design Smells (max: Critical)

| Critical | Major | Minor |
|---|---|---|
| Silent error on user-facing action (catch block with no toast/feedback on form submit, button click, data fetch) | Multiple primary buttons in same view | Spacing drift (inconsistent margin/gap between same-level siblings) |
| Missing error prevention — no validation on user input or no confirmation on destructive action | Blank panel (conditional render to null/empty div without `EmptyState` component) | Color overload (3+ status colors in same section) |
| — | Missing loading state on async action handler | Typography soup (3+ distinct text sizes in one component) |
| — | — | Orphaned icon (no tooltip/aria-label) |
| — | — | Overflow without escape (truncation with no tooltip/expand) |

Silent errors on background operations (analytics, logging, prefetch) are **Acceptable** — do not flag.

#### 3. Error Handling (max: Critical)

| Critical | Major | Minor |
|---|---|---|
| Silent failure on user-facing action — catch block logs/resets but shows nothing to user | Unguarded destructive action (delete/remove with no confirmation) | Generic error message ("Something went wrong") instead of specific |
| Data loss — user input not preserved on failed submit (form state reset in catch block) | Missing client-side validation on form (server-only validation) | Missing retry mechanism on non-critical fetch |
| Unhandled promise rejection on user-facing operation | Missing error state on primary data fetch (error state not handled) | — |
| Missing error prevention — no validation, no confirmation on destructive action, no disabled state | — | — |

#### 4. Journey Completeness (max: Critical)

| Critical | Major | Minor |
|---|---|---|
| Workflow blocked — user cannot complete the core task (dead end, broken navigation, unreachable state) | Missing empty state on primary view (API returns 0 items, blank screen) | Missing partial-data fallback on secondary field (null avatar, missing description) |
| — | Missing error state on primary data fetch | Missing boundary handling (very long text, max items) |
| — | Missing loading state (initial load shows nothing) | — |
| — | Missing success/confirmation feedback after completing action | — |

#### 5. Heuristic Evaluation (max: Major — with one Critical override)

| Critical (override) | Major | Minor |
|---|---|---|
| No error prevention — no validation on user input or no confirmation on destructive action *(cross-cutting override, cross-ref Error Handling)* | No visibility of system status (no loading indicator, no progress, no feedback after action) | Missing keyboard shortcut for power-user action |
| — | No user control/freedom (can't go back, can't cancel, no exit path) | Missing smart default that could reduce effort |
| — | Terminology mismatch (labels don't match user's mental model) | Missing tooltip or helper text |
| — | No recognition support (options hidden, context lost across steps) | Minor aesthetic clutter (extra element that could be removed) |
| — | Error messages not actionable (no guidance on what to do) | — |

**Rule:** Heuristic violations are Major except for missing error prevention, which is always Critical per the cross-cutting override.

#### 6. Gestalt & Visual Hierarchy (max: Critical)

| Critical | Major | Minor |
|---|---|---|
| Visual drift from design system — component looks nothing like its design system counterpart due to custom overrides | Weak hierarchy — no clear reading order, primary CTA doesn't stand out | Minor proximity issue — slightly inconsistent spacing between groups |
| Broken figure-ground — modal/overlay has no backdrop or doesn't de-emphasize background (missing design system `Dialog` pattern) | Similarity violation — same-function elements styled differently | — |
| — | Screen looks like it belongs to a different product vs reference feature | — |

**Rule:** Critical only when it traces to design system drift. Subjective hierarchy/layout opinions are Major or Minor.

#### 7. Cross-Feature Consistency (max: Major)

| Major | Minor |
|---|---|
| Different state handling pattern than reference feature (custom loading vs `Skeleton`) | Different file organization than reference |
| Different component choice for same UI need — *if this violates the design system, flag under UI Consistency as Critical instead* | Minor visual density difference |
| Different navigation pattern (missing breadcrumb, different back behavior) | — |

#### 8. Responsive (max: Major)

| Major | Minor |
|---|---|
| Content overflow/overlap at 768px — element unusable or obscured | Minor truncation that doesn't block functionality |
| Interactive element (dropdown, modal, popover) not fully visible at tablet width | Slight reflow that doesn't break hierarchy |
| No scroll/collapse strategy for wide content (table, grid) at tablet | — |
| Layout breaks — elements stack in wrong order, hierarchy lost | — |

#### 9. Content & Microcopy (max: Major)

| Major | Minor |
|---|---|
| Misleading copy — label/description contradicts what the UI actually does | Generic button label ("Submit", "OK") instead of specific action verb |
| Error message gives no guidance — user doesn't know what went wrong or what to do | Placeholder text not helpful |
| Confirmation dialog unclear about consequences of action | Terminology inconsistency (same concept, different name across 3+ files) |
| — | Hardcoded copy that should be externalized for i18n |

**Rule:** Copy is never Critical, even if misleading.

#### 10. Efficiency (max: Major)

| Major | Minor |
|---|---|
| Redundant round-trip — navigated away and back for something that could be inline | Could pre-fill a field from existing data but doesn't |
| Core workflow takes 2x+ more clicks than necessary vs reference feature | Multi-step flow forces linear sequence where jump-between would be safe |
| Cognitive load — multiple moments of confusion during walkthrough | Single moment of hesitation during walkthrough |

---

## INPUTS

Read the following before starting:

1. **Product context:** Look for a product context file in this order:
   a. `.claude/context/product.md`
   b. `.claude/product.md`
   c. `PRODUCT.md` in the repo root
   d. If none found, ask the user: "I need some product context for a thorough review. Please tell me:
      - What is the product and who are its users?
      - Where is the design system / component library? (e.g., `src/components/ui/`, a Storybook instance, shadcn/ui, MUI, Chakra, etc.)
      - What CSS framework/tokens are used? (Tailwind, CSS Modules, CSS-in-JS, CSS custom properties, etc.)
      - What are 2-3 existing features I can use as reference for consistency checks?"
2. **Branch diff:** Run `git diff main...HEAD --name-only` to understand what files changed in this branch. If `main` doesn't exist, try `master`, then ask the user for the base branch name.
3. **Component library:** Read the design system / component library path from product context to understand the existing component inventory.

---

## PHASE 0: DISCOVERY — Feature Map & Flow

Before reviewing anything, build a complete picture of the feature.

### 0a. Map the feature tree

Starting from the changed files in the branch diff:

1. Identify all UI-relevant files: components, pages, layouts, styles, hooks that drive UI state.
2. For each component, read its imports to determine:
   - Which child components does it render?
   - Which design system components does it use?
   - Which shared components (from outside the feature directory) does it import?
3. If the feature imports components from outside its directory, include those files in scope. Follow imports one level deep outside the feature boundary.

### 0b. Discover the user flow

Determine if this is a multi-step flow or a single screen:

1. Read the main container or entry point. Look for:
   - Route definitions mapping URL paths to components
   - State machines or discriminated unions that switch between views
   - Step/wizard patterns with next/back navigation
   - Conditional renders that show different screens based on state
2. If a flow exists, map it:
   - Which step comes first?
   - What triggers transitions between steps?
   - Are there branches (e.g., path A vs path B)?
   - Can the user go back? Skip steps?
   - Where does the flow terminate?
3. For each step/screen, identify: the component rendered, the hook(s) driving it, the user action that advances to the next step, and what data carries forward.

If no flow is found (single screen), note it and skip flow mapping.

### 0c. Identify the closest existing feature

From the Reference Features listed in product context, identify the existing feature that uses the most similar screen pattern. This will be the baseline for cross-feature consistency checks in later phases. If no reference features are documented, ask the user to name 1-2 existing features that represent good patterns in this codebase.

### 0d. Output: Scope summary

Before proceeding, produce a brief summary:

```
Feature: <name>
Type: <single screen / multi-step flow / list+detail / etc.>
Files in scope: <count> (<count> in feature + <count> shared)
Closest existing feature for comparison: <path>

Flow (if multi-step):
  Step 1: <Screen> (<file>)
    → <action> → Step 2
  Step 2: <Screen> (<file>)
    → <action> → Step 3
    ← back → Step 1
  ...

Workflows to test:
  W1: <workflow name> — <brief description of end-to-end task>
  W2: <workflow name> — <brief description>
  ...
```

Wait for user confirmation before proceeding.

---

## PHASE 1: CLARIFICATION

Based on the discovery, ask the user (in a single batch):
1. Is my understanding of the feature scope correct? [state what you found]
2. Are there specific areas you're concerned about or want me to focus on?
3. Any constraints I should know? (deadline pressure, known tech debt, phased rollout, etc.)
4. Does the dev environment have sample data or do I need to create it?

Wait for confirmation, then proceed.

---

## PHASE 2: CODEBASE ANALYSIS & WALKTHROUGH TESTING

### Part A: Understand the existing patterns (code)
- Read the reference feature identified in Phase 0 to establish the baseline: how it handles errors, loading states, empty states, forms, navigation
- Note the patterns used so you can compare the branch code against them

### Part B: Understand the new/changed code
- Read all UI-relevant files from the branch diff
- Trace the user journey through the code: entry point → intermediate steps → completion/exit
- Identify all user-facing states: default, loading, empty, error, success, edge cases

### Part C: Data setup (via Claude in Chrome)

Before taking any screenshots or testing workflows, ensure the UI has real data:

1. **Start the dev server** — identify the start command from `package.json` scripts or `README`. Start it if not already running.
2. **Navigate to the feature** — using Claude in Chrome, go to the entry point of the flow.
3. **Check for prerequisite data** — does this feature need sample data to be meaningful? If the page is empty/default, create 2-3 realistic sample items using the UI or seed scripts.
4. **Populate the reference feature too** — if the reference feature page (from Phase 0c) is also empty, create sample data there. Comparing two empty pages is meaningless.

The goal: review a POPULATED UI, not empty states.

### Part D: Walkthrough testing (via Claude in Chrome)

Execute every workflow identified in Phase 0d end-to-end in the browser. This is the most important part of the review.

For each workflow:

1. **Perform it step by step** — type real inputs, click real buttons, wait for real responses.
2. **Count every click, input, and navigation** — each deliberate user action is a "step."
3. **Screenshot each state transition** — before the action and after the action.
4. **Note moments of confusion** — where did you hesitate? Where was the next step not obvious? Where did you have to guess?
5. **Record feedback** — what did the UI tell you after each action? (toast, spinner, redirect, nothing?)

Then test error paths live:
- **Submit with empty/invalid inputs** — what happens? Screenshot the result.
- **Try actions without prerequisites** — e.g., submitting without selecting required options. Screenshot the result (or lack of feedback).
- **Try boundary cases** — very long text, max items, zero items. Screenshot anything that breaks.

Then test cross-feature transitions:
- If the feature hands off to another feature (e.g., save → view elsewhere), **follow the full path**.
- Screenshot the destination and verify data arrived correctly.

### Part E: Viewport screenshots

Only AFTER completing walkthrough testing, capture responsive behavior:

1. **Desktop (1440px):** Re-run the core happy path, screenshot each screen.
2. **Small desktop (1024px):** Resize, re-run the core happy path, screenshot each screen.
3. **Tablet (768px):** Resize, re-run the core happy path, screenshot each screen. At this width, also trigger all interactive elements (dropdowns, modals, popovers) and screenshot their open state.

Navigate to 2-3 pages of the reference feature and screenshot them at desktop width for visual comparison (with real data populated).

---

## PHASE 3: UX REVIEW

Structure your review as follows. Skip sections that don't apply, but state what you skipped.

Every finding must be classified as one of:
- **Code-verified** (🔍) — found by reading source code, definitively correct
- **Visually-verified** (👁️) — found by inspecting screenshots of populated UI, definitively correct
- **Interaction-verified** (🖱️) — found by executing a workflow in the browser, definitively correct
- **Judgment call** (💭) — subjective assessment, may need team discussion

Every finding must include a **file:line reference** (for 🔍), **screenshot reference** (for 👁️), or **workflow step description** (for 🖱️).

---

### 1. Flow Summary
- Describe the user journey as implemented in the code
- List all screens/views/states in order
- Note entry points and exit points

---

### 2. Heuristic Evaluation (Code-verified 🔍)

Evaluate against Nielsen's 10 heuristics by reading the code:
- **Visibility of system status** — Are loading states, progress indicators, and feedback implemented?
- **Match between system and real world** — Do labels, copy, and terminology make sense for the target user?
- **User control and freedom** — Can users go back, cancel, undo? Are exit paths clear?
- **Consistency and standards** — Does this follow the patterns already in the codebase? (See Phase 2 baseline)
- **Error prevention** — Are there input validations, confirmations for destructive actions, disabled states where needed?
- **Recognition over recall** — Are options visible? Are defaults sensible? Is context preserved across steps?
- **Flexibility and efficiency** — Keyboard shortcuts, bulk actions, smart defaults for power users?
- **Aesthetic and minimalist design** — Is the UI focused? Any unnecessary elements or information overload?
- **Error recovery** — Are error messages specific and actionable? Can users recover without losing work?
- **Help and documentation** — Tooltips, helper text, onboarding cues where needed?

For each, state: ✅ Pass / ⚠️ Warning / ❌ Fail — with specific file:line evidence.

---

### 3. UI Consistency Audit (Code-verified 🔍)

#### How to verify
1. **Build a component inventory from the branch** — List every UI component imported or created in the changed files. For each, check if an equivalent already exists in the design system / component library path (from product context).
2. **Search for hardcoded values** — Grep the branch diff for hardcoded colors (hex, rgb, hsl), pixel values for spacing/sizing, and font properties. Compare against token/theme files.
3. **Pattern match against the reference feature** — Read the reference feature identified in Phase 0. Note how it handles the same UI patterns (modals, forms, tables, navigation).

#### Checklist
- **Component reuse** — Is the branch using existing design system components, or are there custom one-offs that duplicate existing components? Flag any new component that overlaps with an existing one. [file:line for each]
- **Spacing and layout** — Are the same spacing tokens/conventions used as the rest of the product? Flag any hardcoded `px`, `rem`, `em` values that should be tokens. [file:line for each]
- **Typography** — Are heading levels, font sizes, and weights consistent with existing pages? Flag any inline font styles or non-token type values. [file:line for each]
- **Color usage** — Are colors coming from tokens/theme, or are there hardcoded values? Flag every hardcoded color with the file and line. [file:line for each]
- **Interaction patterns** — Do modals, toasts, dropdowns, form controls use the same components and behave the same way as elsewhere in the product? Flag any custom implementation of a pattern that already has an established component. [file:line for each]
- **Naming conventions** — Are CSS classes, component names, and props following the existing codebase conventions? Check casing, prefixing, and file/folder structure. [file:line for each]

---

### 4. Cross-Feature Consistency (Code-verified 🔍)

Compare the branch code structurally against the reference feature from Phase 0.

#### How to verify
1. Read the reference feature's file structure, state handling, component usage, and visual density.
2. Compare each dimension against the new branch code.

#### Checklist
- **File structure** — Does the new feature follow the same organizational pattern as the reference?
- **State handling** — Same approach to loading, error, empty states?
- **Component choices** — For similar UI needs, does the new feature pick the same components?
- **Visual density** — Is the spacing, padding, and information density consistent with the reference?
- **Navigation patterns** — Back buttons, breadcrumbs, tab structures consistent?

Flag deviations with file:line references from both the new code and the reference.

---

### 5. Journey Completeness (Code-verified 🔍 + Interaction-verified 🖱️)

#### How to verify
1. **Identify all data sources** — Find every API call, query, or data fetch in the branch (e.g., `fetch`, `axios`, `useQuery`, `getServerSideProps`, API route handlers, or equivalent patterns in your framework). Each data source is a point where the UI can branch into multiple states.
2. **Trace conditional renders** — For each component that consumes data, look for conditional rendering (ternary operators, `if/else`, early returns, `switch` cases, `&&` guards, or equivalent patterns). Each condition should map to a UI state below.
3. **Cross-reference with existing patterns** — Check how the reference feature handles these states. If the codebase has `<EmptyState />`, `<Skeleton />`, `<ErrorBoundary />`, or similar components, the branch should use the same patterns.
4. **Check user action outcomes** — For every user action (form submit, button click, toggle, delete), trace what happens on success AND failure. Both paths should have explicit UI handling.
5. **Verify live** — During walkthrough testing (Phase 2D), confirm which states you actually triggered. Mark each state as "tested live" or "code-only."

#### State checklist
For each page/view/component in the flow, verify:

- [ ] **Default / happy path** — The primary rendered state with data present
- [ ] **Loading — initial** — What the user sees before the first data load completes (skeleton, spinner, etc.)
- [ ] **Loading — action pending** — What the user sees after triggering an action (button disabled + spinner, optimistic update, etc.)
- [ ] **Empty state** — What renders when the API returns success but with zero items / no data. Check: is there a message, illustration, and a CTA to guide the user?
- [ ] **Error — API/network failure** — What renders when the fetch itself fails (500, timeout, offline). Check: is there a retry mechanism?
- [ ] **Error — validation** — What renders for invalid user input. Check: are errors shown inline next to the field, or only as a generic toast?
- [ ] **Error — permission denied** — What renders if the user lacks access (403). Check: is there a meaningful message or just a blank screen?
- [ ] **Success / confirmation** — What renders after a successful action (toast, redirect, inline confirmation). Check: is it clear what happened and what to do next?
- [ ] **Partial data** — What renders when some fields are null/undefined (e.g., user has no avatar, description is missing). Check: are there fallbacks or does the layout break?
- [ ] **Boundary cases** — What happens at extremes: 0 items, 1 item, max items, very long text, very long lists. Check: does the UI truncate, paginate, virtualize, or break?

For each missing state, flag the specific component and file:line, and note which existing codebase pattern should be used to implement it.

#### State coverage matrix

Output a summary table:

```
| Screen/View       | Default | Loading | Empty | Error | Success | Partial | Tested Live? |
|-------------------|---------|---------|-------|-------|---------|---------|--------------|
| [Screen 1]        | ✅      | ✅      | ❌    | ⚠️    | ✅      | ❌      | ✅ W1, W2    |
| [Screen 2]        | ✅      | ❌      | n/a   | ❌    | ✅      | ✅      | ❌ code only |
```

---

### 6. Efficiency of the Journey (Code-verified 🔍 + Interaction-verified 🖱️)

#### How to verify (code)
1. **Map the interaction sequence** — Starting from the entry point component, trace every user action (click, input, navigation) required to complete the core task. Each action that requires a deliberate user decision or changes the view counts as a "step."
2. **Compare against the shortest possible path** — Ask: can any two steps be combined? Can any step be eliminated with a smart default or inline action?
3. **Check for round-trips** — Look for patterns where the user is sent to another page/modal and then brought back (navigate away → do something → return). These are often collapsible.

#### How to verify (live)
4. **Use actual click counts from Phase 2D walkthrough** — Report MEASURED clicks alongside code-estimated clicks. Flag any discrepancy.
5. **Note cognitive load moments** — Where did you hesitate or search for the next action during the walkthrough? Where was the next step not obvious?
6. **Compare code vs experience** — Does the code suggest N steps but the live experience feels like more (hidden sub-steps, unexpected modals) or fewer (smart defaults that skip steps)?

#### Checklist
- How many steps/clicks does the core task take? (State both code-estimated AND measured numbers)
- Are there redundant steps that could be collapsed?
- Is the information asked for in a logical order?
- Could smart defaults reduce user effort? (Check if any form fields could be pre-filled from existing data)
- Are there opportunities for inline editing vs. navigate-away-and-back?
- For multi-step flows: can the user jump between steps or are they forced into a linear sequence?
- Were there moments during the walkthrough where the next action was unclear? [describe each]

---

### 7. Design Smell Scan (Code-verified 🔍)

A fast-pass checklist for common anti-patterns. These are quick to verify from code and frequently missed.

#### How to verify
Scan all UI-relevant files in the branch for these patterns. Each smell is a grep or a quick read — don't deep-analyze, just flag.

| Smell | What to search for |
|-------|-------------------|
| **Multiple primary buttons** | More than one `type="primary"` or `variant="primary"` in the same component/view |
| **Blank panels** | Conditional renders where `length === 0` results in `null`, empty `<div>`, or no render — without an `EmptyState` component |
| **Silent errors** | `catch` blocks that reset state or log but show no user-visible feedback (no toast, no error component, no inline message) on **user-facing actions** |
| **Silent errors (background)** | `catch` blocks on analytics, logging, or non-user-facing operations that silently fail — **Acceptable, do not flag** |
| **Missing error prevention** | No validation on user input or no confirmation on destructive actions |
| **Missing loading** | Async action handlers (`onClick` that calls an API) where the trigger button has no loading/disabled state |
| **Spacing drift** | Same-level siblings with different margin/gap values instead of consistent gap on parent |
| **Color overload** | 3+ semantic/status colors used in the same row or small section |
| **Typography soup** | 3+ distinct text sizes in one component or section |
| **Orphaned icon** | Icon-only button without a tooltip or aria-label |
| **Overflow without escape** | Truncated content (`truncate`, `overflow-hidden`, `text-ellipsis`) with no tooltip, expand, or other way to see the full value |

Flag each with file:line. **Use the Severity Rubric (Design Smells section) to assign severity.** Do not invent severity levels — look up each smell in the rubric.

---

### 8. Gestalt & Visual Hierarchy (Visually-verified 👁️)

This section requires visual verification via Claude in Chrome. Do NOT evaluate from code alone.

**IMPORTANT:** Screenshots must be from POPULATED states (with real data from Phase 2C), not empty/default states. Comparing two empty pages is meaningless.

#### How to verify
1. **Use the screenshots captured during walkthrough testing (Phase 2D)** at desktop viewport width — these already show the UI with real data and in mid-workflow states.
2. **Visually inspect each screenshot** against the checklist below.
3. **Compare with reference feature screenshots** captured in Phase 2E (also with real data populated).

#### Checklist
- **Proximity** — Are related elements visually grouped together? Is there enough spacing between unrelated sections to create clear separation?
- **Similarity** — Do elements that serve the same function look the same? (e.g., all primary CTAs same style, all section headers same weight/size)
- **Hierarchy** — Is there a clear visual reading order? Does the most important element draw attention first? Is there a logical top-down or left-right flow?
- **Figure-ground** — Are modals, popovers, and overlays clearly layered above the background? Is the backdrop visible and does it de-emphasize background content?
- **Consistency with existing product** — Does this screen look like it belongs in the same product as the reference feature pages?

Each finding should reference the specific screenshot and describe what's visible.

---

### 9. Content & Microcopy (Code-verified 🔍)

#### How to verify
1. **Extract all user-facing strings** — Search the branch diff for all visible text: button labels, headings, placeholder text, error messages, toast messages, tooltip content, helper text, empty state messages, confirmation dialogs. Include strings in constants files, i18n files, and inline JSX/templates.
2. **Categorize by type** — Group the strings into: CTAs, labels, error messages, helper text, placeholders, confirmations. Review each group for consistency.
3. **Check against existing copy patterns** — Read similar flows in the codebase to see how they word things. The new branch should match the existing voice and terminology.

#### Checklist
- Are button labels specific and action-oriented ("Save changes", "Invite member") or generic ("Submit", "OK", "Click here")? [file:line for each generic label]
- Are error messages actionable — do they tell the user what went wrong AND what to do about it? Or are they generic ("Something went wrong")? [file:line for each]
- Are placeholder texts helpful hints or do they disappear and leave the user guessing what the field wants? [file:line for each]
- Is terminology consistent with the rest of the product? (Only flag if you find the same concept called something different in 3+ other files in the codebase) [file:line for each]
- Is there any hardcoded copy that should be externalized for i18n? [file:line for each]
- Are confirmation dialogs clear about the consequences of the action? [file:line for each]

---

### 10. Error Handling & Recovery (Code-verified 🔍 + Interaction-verified 🖱️)

#### How to verify (code)
1. **Find all error boundaries** — Search the branch for `try/catch`, `.catch()`, `onError`, error callback props, `ErrorBoundary` components, and query/mutation error states (e.g., `isError` from React Query/TanStack Query, error states from SWR, Redux Toolkit Query, Apollo GraphQL, or the project's data-fetching pattern).
2. **Classify each error handler** — Is this a user-facing action (form submit, button click, data fetch that populates the UI) or a background operation (analytics, logging, prefetch)? This determines severity.
3. **Trace each user-facing error path to UI** — For every error catch on a user-facing action, follow the code to see what the user actually sees. Does it render an error component? Show a toast? Silently log and show nothing?
4. **Test destructive actions** — Identify all delete, remove, overwrite, or irreversible actions. Check if each one has a confirmation step and/or undo mechanism.
5. **Check form validation** — For every form, look at validation logic. Is it client-side (before submit), server-side (after submit), or both? Are errors shown inline per field or only as a summary?

#### How to verify (live — do not skip this)
6. **Trigger errors via Claude in Chrome** — Don't just trace error paths in code. Actually cause them:
   - Submit forms with missing required fields. Screenshot the result.
   - Try actions without selecting prerequisites. Screenshot the result.
   - Try invalid inputs (empty strings, special characters, very long text). Screenshot the result.
   - For each error triggered, note: did the UI give feedback? Was the feedback helpful? Was user input preserved?
7. **Compare code prediction vs live result** — For every error handler found in code, note whether you were able to trigger it live and whether the actual behavior matched what the code suggested.

#### Checklist
- How are API errors on user-facing actions handled? (Caught and shown vs. silent failure vs. crash) [file:line for each] — **Silent failure on user-facing actions is Critical severity**
- How are API errors on background operations handled? (Silent failure is acceptable here) [file:line, for awareness only]
- Are form validations client-side, server-side, or both? [file:line]
- Can users retry failed actions without re-entering data? [file:line]
- Is user input preserved on error? (Check if form state is reset on failed submit) [file:line]
- Are destructive actions guarded (confirm dialog, undo, soft delete)? [file:line for each unguarded destructive action]
- Are there any unhandled promise rejections or missing `.catch()` blocks on user-facing operations? [file:line for each]

---

### 11. Responsive Behavior (Visually-verified 👁️ + Interaction-verified 🖱️)

This section requires visual AND interactive verification via Claude in Chrome. Do NOT guess responsive behavior from code alone.

#### How to verify
1. **Use the screenshots captured in Phase 2E** at all three viewport widths (1440px, 1024px, 768px).
2. **Compare across widths** — For each screen, compare the three screenshots. Look for layout shifts, content reflows, and any visual breakage.
3. **Re-run the core happy path at 768px** — Don't just screenshot the static page. Actually click through the main workflow at tablet width to find interactive element issues that only appear during use.
4. **Test interactive elements at tablet width** — At 768px, use Claude in Chrome to open/trigger all interactive elements (dropdowns, modals, popovers, tooltips, sidebars) and screenshot their open state.
5. **Compare against reference feature** — Check how similar existing pages in the product behave at the same widths. The new flow should follow the same responsive strategy.

#### Checklist
- Does the layout adapt gracefully at each breakpoint without breaking?
- Is content reflow logical — does information hierarchy hold at narrower widths?
- Are interactive elements (dropdowns, modals, popovers) fully visible and usable at tablet width?
- Do tables, data grids, or wide content areas have a scroll/collapse strategy at smaller widths?
- Is the responsive behavior consistent with how the rest of the product handles narrower viewports?
- Flag anything that overflows, overlaps, truncates unexpectedly, or becomes unusable [screenshot reference for each]

---

## PHASE 4: REVIEW OUTPUT

Structure the output as: **verdict first, details second.** The reviewer should know the outcome and what to fix within the first 10 lines.

---

### 1. Verdict & Score (always first)

Compute the score using the Severity Rubric:

```
## Verdict: [Approve / Request changes / Flag for designer]
## Score: [X/100] (or "Auto-fail" if any Criticals)

Critical findings: [count] — if > 0, verdict is auto-fail "Request changes"
Major findings: [count] × -5 = -[total]
Minor findings: [count] × -2 = -[total]
Score: 100 - [major deductions] - [minor deductions] = [score]

Verdict rules:
- Any Critical → Request changes (auto-fail, score not shown)
- Score >= 90 → Approve
- Score 80-89 → Request changes (fixable, must resolve before merge)
- Score < 80 → Request changes (significant rework needed)
- Subjective/judgment-only findings → Flag for designer
```

### 2. Action Items Table

```
| # | Severity | Dimension | Type | File:Line | Finding | Recommendation |
|---|----------|-----------|------|-----------|---------|----------------|
| 1 | Critical | UI Consistency | 🔍 | card.tsx:8 | Hardcoded #f5f5f5 — should use bg-muted | Replace with design system token |
| 2 | Critical | Error Handling | 🖱️ | W1 step 3 | Clicked submit without required field — no feedback | Add validation feedback |
| 3 | Major | Journey | 🔍 | list.tsx:12 | Empty state renders blank div | Use EmptyState component |
| 4 | Major | Responsive | 👁️ | screenshot:tablet | Modal overflows at 768px | Check max-width constraint |
| 5 | Minor | Content | 🔍 | header.tsx:22 | Generic "Submit" button label | Use specific action verb |
```

### 3. Score Card

```
| Dimension                  | Criticals | Majors | Minors | Deduction |
|----------------------------|-----------|--------|--------|-----------|
| Heuristic evaluation       | 0         | 0      | 0      | 0         |
| UI consistency             | 0         | 0      | 0      | 0         |
| Cross-feature consistency  | 0         | 0      | 0      | 0         |
| Journey completeness       | 0         | 0      | 0      | 0         |
| Efficiency                 | 0         | 0      | 0      | 0         |
| Design smells              | 0         | 0      | 0      | 0         |
| Gestalt & hierarchy        | 0         | 0      | 0      | 0         |
| Content & microcopy        | 0         | 0      | 0      | 0         |
| Error handling             | 0         | 0      | 0      | 0         |
| Responsive (tablet)        | 0         | 0      | 0      | 0         |
| **Total**                  | **0**     | **0**  | **0**  | **0**     |
```

### 4. Walkthrough Summary

```
| Workflow | Steps/Clicks | Completed? | Issues Found |
|----------|-------------|------------|--------------|
| W1: ... | N | ✅/❌ | ... |
```

### 5. Scope (from Phase 0)

```
Feature: <name> (<count> files)
Type: <pattern>
Reference feature: <path>
Flow: <step summary>
```

### 6. State Coverage Matrix

```
| Screen/View       | Default | Loading | Empty | Error | Success | Partial | Tested Live? |
|-------------------|---------|---------|-------|-------|---------|---------|--------------|
| [Screen 1]        | ✅      | ✅      | ❌    | ⚠️    | ✅      | ❌      | ✅ W1, W2    |
| [Screen 2]        | ✅      | ❌      | n/a   | ❌    | ✅      | ✅      | ❌ code only |
```

### 7. Detailed Findings (from Phase 3)

All findings in a single table, grouped by severity.

```
| # | Phase | Type | Severity | File:Line | Section | Finding | Recommendation |
|---|-------|------|----------|-----------|---------|---------|----------------|
| 1 | Code  | 🔍   | Critical | auth.tsx:45 | Error handling | Silent catch on user action | Add toast |
| 2 | Live  | 🖱️   | Critical | W1 step 3 | Error handling | Submit without required field = no feedback | Add validation |
| 3 | Code  | 🔍   | Major    | list.tsx:12 | Journey completeness | Missing empty state | Add EmptyState |
| 4 | Visual| 👁️   | Major    | screenshot:tablet | Responsive | Modal overflow at 768px | Constrain width |
| 5 | Code  | 🔍   | Minor    | card.tsx:8 | UI consistency | Hardcoded color | Use token |
| 6 | —     | 💭   | Minor    | header.tsx:22 | Gestalt | Hierarchy unclear | Increase contrast |
```

### 8. Quick Wins
3-5 issues that are low-effort to fix right now.

### 9. Pre-Merge Blockers
All Critical and Major severity findings — these should be resolved before the PR merges.

### 10. Follow-Up Recommendations
Improvements worth doing in a subsequent PR.

---

## PHASE 5: POST TO PR

After outputting the review in the conversation, post it as a comment on the current branch's pull request so the review is permanently recorded.

### 5a. Detect the PR

Run: `gh pr view --json number,url --jq '.number, .url'`

- If a PR exists → continue to 5b.
- If no PR exists (command fails or returns nothing) → inform the user: "No open PR found for this branch. Would you like to create one first, or skip posting the comment?" Wait for their response before proceeding.

### 5b. Format the PR comment

Structure the comment so the verdict is immediately visible and the full report is collapsible:

```markdown
## UX Review: [Feature Name]

### Verdict: [Approve / Request changes / Flag for designer]
### Score: [X/100] (or "Auto-fail — [N] Critical findings")

| # | Severity | Dimension | Type | File:Line | Finding | Recommendation |
|---|----------|-----------|------|-----------|---------|----------------|
| ... (action items table from Phase 4 section 2) |

<details>
<summary>Full Review Report</summary>

(Include all remaining sections from Phase 4: Score Card with per-dimension deductions, Walkthrough Summary, Scope, State Coverage Matrix, Detailed Findings, Quick Wins, Pre-Merge Blockers, Follow-Up Recommendations)

</details>
```

### 5c. Post the comment

Run: `gh pr comment <number> --body "<formatted comment>"`

Use a HEREDOC to pass the body to avoid shell escaping issues:
```bash
gh pr comment <number> --body "$(cat <<'EOF'
<comment content>
EOF
)"
```

Confirm to the user that the comment was posted and include the PR URL.

$ARGUMENTS
