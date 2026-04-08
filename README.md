# ux-review

Automated UX reviews for Claude Code. Reads your code, captures screenshots via Claude in Chrome, and executes workflows end-to-end — catching issues before they reach users.

## What It Does

A senior UX engineer in your terminal. It reviews the **actual implemented UI** — not design files — by combining three verification modes:

- **Code analysis** — reads source files to check structure, patterns, and design system usage
- **Visual inspection** — captures screenshots at multiple viewports to verify rendered output
- **Interactive walkthrough** — executes every workflow end-to-end, clicking buttons, filling forms, triggering errors

Reviews are scored across **10 dimensions**: UI consistency, design smells, error handling, journey completeness, heuristic evaluation, visual hierarchy, cross-feature consistency, responsive behavior, content & microcopy, and efficiency.

## Prerequisites

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) (CLI, desktop app, or IDE extension)
- [Claude in Chrome](https://chromewebstore.google.com/detail/claude-in-chrome/) extension — installed, active, and connected
- [`gh` CLI](https://cli.github.com/) — for posting review comments to PRs
- A running dev server (localhost)
- A feature branch with UI changes

## Installation

```bash
claude /plugin install ux-review
```

Or for local development:

```bash
claude --plugin-dir /path/to/ux-review
```

## Setup (Recommended)

For best results, create a product context file at `.claude/context/product.md` in your project. See [`examples/product-context.example.md`](examples/product-context.example.md) for a template.

This tells the skill about your design system, component library, CSS tokens, and reference features — enabling more accurate consistency checks.

The skill works without this file — it will ask you for the needed context interactively — but having it pre-written saves time on every review.

## Usage

In Claude Code, type:

```
/ux-review [optional: feature name or description]
```

The skill will:
1. **Discover** — map your feature from the branch diff, identify flows and reference features
2. **Clarify** — confirm scope and ask about focus areas
3. **Analyze & Test** — read code, set up test data, execute all workflows in the browser
4. **Review** — evaluate across 10 dimensions with evidence-based findings
5. **Report** — produce a scored verdict with action items
6. **Post to PR** — automatically post the review as a PR comment

## Review Dimensions

| # | Dimension | Max Severity | What It Checks |
|---|-----------|-------------|----------------|
| 1 | UI Consistency | Critical | Design system compliance, component reuse, token usage |
| 2 | Design Smells | Critical | Anti-patterns: silent errors, blank panels, missing loading |
| 3 | Error Handling | Critical | Error prevention, recovery, validation, destructive actions |
| 4 | Journey Completeness | Critical | State coverage: loading, empty, error, success, partial, boundary |
| 5 | Heuristic Evaluation | Major* | Nielsen's 10 heuristics (*Critical override for error prevention) |
| 6 | Gestalt & Visual Hierarchy | Critical | Proximity, similarity, reading order, figure-ground |
| 7 | Cross-Feature Consistency | Major | Patterns, components, navigation vs reference feature |
| 8 | Responsive | Major | Layout at 1440px, 1024px, 768px; interactive elements at tablet |
| 9 | Content & Microcopy | Major | Button labels, error messages, terminology, confirmation clarity |
| 10 | Efficiency | Major | Click counts, cognitive load, smart defaults, round-trips |

## Scoring

- Start at **100 points**
- Any **Critical** finding = auto-fail (blocks merge)
- **Major** = -5 points per finding
- **Minor** = -2 points per finding
- **>= 90**: Approve | **80-89**: Request changes | **< 80**: Significant rework

## Customization

### Product Context

Create `.claude/context/product.md` in your project with:
- Product description and target users
- Design system / component library path
- CSS framework and token locations
- 2-3 reference features for consistency baselines

See [`examples/product-context.example.md`](examples/product-context.example.md) for a full template.

### What the skill adapts to your project

- Component library inventory (discovered from your codebase)
- CSS tokens (discovered from your theme files)
- Reference features (from your product context or asked interactively)
- Data-fetching patterns (React Query, SWR, Apollo, or whatever you use)
- Base branch (tries `main`, then `master`, then asks)

## License

MIT
