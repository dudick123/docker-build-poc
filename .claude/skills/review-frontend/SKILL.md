---
name: review-frontend
description: >
  Senior Frontend Developer review of an OpenSpec proposal or implemented React/TypeScript code.
  Examines React architecture, TypeScript strictness, React Query configuration, accessibility (WCAG AA),
  bundle performance, error boundaries, component patterns, and testing infrastructure. Use when working
  on frontend proposals, React component changes, or when the user asks for a frontend review.
---

# Senior Frontend Developer Review

Perform a deep technical review from the perspective of a Senior Frontend Developer. Examine either an OpenSpec proposal (proposal.md, design.md, tasks.md, specs/) or implemented React/TypeScript code. Validate against `docs/TECH-STANDARDS.md` section 2.

## When to Use

- A frontend module proposal is ready for review
- React/TypeScript code changes need review
- The user asks for a "frontend review", "React review", or "UI review"
- The `/review` router delegates to this skill

## Execution Steps

### Step 1: Identify the target

Determine what to review:
- If the user specifies a proposal, read `openspec/changes/<proposal-name>/` artifacts (proposal.md, design.md, tasks.md, specs/)
- If the user specifies code, read the relevant TypeScript/React files under `frontend/`
- If neither is specified, check for the most recent proposal in `openspec/changes/`

### Step 2: Read the standards

Read the following from `docs/TECH-STANDARDS.md`:
- Section 2: TypeScript / React Code Style (Frontend) — tsconfig, ESLint, naming, component patterns, React Query, accessibility, performance, security

Also read `CLAUDE.md` for project-level context (team has no prior React experience, auto-generated API clients).

### Step 3: Create the output directory

```bash
mkdir -p docs/tech-review
```

### Step 4: Perform the review

Evaluate the proposal or code against these focus areas:

**TypeScript Configuration (5 Enhanced Strict Flags)**
- `strict: true` in tsconfig.json
- `noUncheckedIndexedAccess` — critical for dashboard consuming external API data
- `exactOptionalPropertyTypes` — catches connector response mismatches
- `noPropertyAccessFromIndexSignature` — forces bracket notation for dynamic keys
- `forceConsistentCasingInFileNames` — prevents cross-platform import bugs
- `verbatimModuleSyntax` — ensures `import type` for type-only imports
- No `any` types except in auto-generated API client code. Use `unknown` + type narrowing.

**ESLint Configuration (6 Required Plugins)**
- `@typescript-eslint` — TypeScript-aware linting
- `eslint-plugin-react-hooks` — Rules of Hooks, dependency arrays
- `eslint-plugin-react` — JSX best practices, prop validation
- `eslint-plugin-jsx-a11y` — accessibility (missing alt, broken ARIA, non-interactive handlers)
- `eslint-plugin-security` — DOM XSS patterns, unsafe innerHTML, regex DoS
- `eslint-plugin-import` — import ordering, no unused imports, no circular dependencies
- No blanket `// eslint-disable` — must specify rule name and justification

**React Query Defaults**
- `staleTime: 30 * 60_000` (30 minutes, matches uniform connector TTL)
- `gcTime: 60 * 60_000` (60 minutes garbage collection)
- `refetchIntervalInBackground: false` on all polling queries — no wasted load from hidden tabs
- Query keys follow hierarchical factory pattern in `query-keys.ts`
- Manual refresh button per module calls `queryClient.invalidateQueries()`

**Error Boundaries**
- Every dashboard module wrapped in `react-error-boundary`
- Connector failure must not crash the entire dashboard (PRD §4.6 Degraded State)
- Fallback shows module name, "data unavailable" message, and retry button
- `resetKeys` tied to project scope selector for automatic recovery
- No stack traces or technical details shown to the user

**Code Splitting and Performance**
- `React.lazy()` + `Suspense` for each dashboard module
- Initial bundle MUST NOT exceed 200KB gzipped. Module chunks under 100KB.
- `@tanstack/react-virtual` for large lists (850+ applications)
- Skeleton loaders (not spinners) for initial data loading
- Background refetches update silently — no loading indicators

**Accessibility (WCAG 2.1 Level AA)**
- Colour alone is insufficient for status — must include text label or icon
- Colour contrast ratios: 4.5:1 normal text, 3:1 large text and UI components
- All interactive elements keyboard-accessible. No tab traps.
- Semantic HTML (`<table>`, `<nav>`, `<main>`, `<section>`) — not div soup
- `aria-live="polite"` for data refresh timestamps and status changes
- Sparkline charts include `aria-label` with trend summary

**Component Patterns**
- No default exports (except `React.lazy()` route components)
- kebab-case file names (`app-status-grid.tsx`)
- One component per file
- Functional components only with explicit return types
- Props defined as dedicated `type` with `readonly` on props and arrays
- No prop spreading (`<Component {...props} />` is banned)

**Testing Infrastructure**
- Vitest as test runner
- React Testing Library (RTL) for component tests
- MSW (Mock Service Worker) for API mocking
- `vitest-axe` for accessibility assertions on module-level components
- `@axe-core/react` in development mode for console accessibility warnings

### Step 5: Write the review

Write the review to `docs/tech-review/{proposal}-frontend-review.md` using this exact format:

```markdown
---
reviewer: Senior Frontend Developer
proposal: <proposal-name>
date: <YYYY-MM-DD>
status: Review Complete
---

# Senior Frontend Developer — Review: <proposal-name>

## Summary

(2-3 sentence overall assessment. Be direct about severity.)

## Critical Findings

(Must-fix items. Use FE- prefix for finding IDs.)

### Finding FE-<N>: <Title>

- **Artifact**: (which file: design.md, tasks.md, spec.md, proposal.md, or source file path)
- **Location**: (section, task number, or line reference)
- **Issue**: (what is wrong — quote the specific TECH-STANDARDS section violated)
- **Impact**: (concrete consequences if not fixed)
- **Recommendation**: (specific fix, not vague guidance)

## Recommendations

(Should-fix improvements. Same structure as findings.)

### Recommendation FE-<N>: <Title>

- **Artifact**:
- **Location**:
- **Issue**:
- **Impact**:
- **Recommendation**:

## Observations

(Nice-to-have notes, minor items, things to watch in future proposals.)

## Standards Compliance

| Standard | Status | Notes |
|----------|--------|-------|
| TECH-STANDARDS §2.1 — tsconfig strict + 5 enhanced flags | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §2.2 — ESLint 6 plugins configured | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §2.3 — No default exports | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §2.4 — kebab-case file names | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §2.5 — Error boundaries per module | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §2.6 — React Query defaults (staleTime/gcTime) | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §2.7 — refetchIntervalInBackground: false | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §2.8 — React.lazy code splitting | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §2.9 — 200KB initial bundle budget | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §2.10 — @tanstack/react-virtual for large lists | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §2.11 — WCAG AA colour + text indicators | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §2.12 — Keyboard navigation | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §2.13 — aria-live for status updates | Met / Partial / Not Met / N/A | |
| TECH-STANDARDS §2.14 — Vitest + RTL + MSW testing | Met / Partial / Not Met / N/A | |
```

### Step 6: Report results

After writing the review file, report to the user:
- Number of critical findings and recommendations
- Top 2-3 most important issues
- Overall standards compliance assessment
- Path to the full review file
