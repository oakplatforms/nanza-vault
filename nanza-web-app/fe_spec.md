# Front End Updates Spec

## Scope
- Frontend-only guidance scoped to `src/`.
- This spec is the source of truth for future Cursor plans/prompts in this repo.
- Routing is read-only reference; do not change it.

## ⚠️ MANDATORY: Before Creating Any Plan

Complete this checklist BEFORE presenting solutions:

- [ ] **Confidence scoring added** - Every solution/decision has 0.0-1.0 score
- [ ] **Alternatives considered** - Listed other approaches with trade-offs
- [ ] **DRY check performed** - Searched for existing patterns/components
- [ ] **Surgical changes verified** - Minimized file count and code deltas
- [ ] **Escalation assessed** - Flagged non-trivial or risky changes
- [ ] **Examples/evidence provided** - Cited specific files/lines

If ANY checkbox is unchecked, the plan is incomplete.

## Engineering Standards and Approach
When working on any task in this codebase:
- **Ask clarifying questions** before starting implementation. Understand requirements fully.
- **Approach problems like a senior engineering team** at Meta or Shopify: prioritize scalability, maintainability, and production-ready code.
- **Break complex problems into smaller pieces**: decompose difficult tasks into manageable, reviewable chunks.
- **Peer review your solutions**: check your approach against what senior developers would recommend. Consider alternatives, edge cases, and long-term maintainability.
- **⚠️ CRITICAL: Confidence Scoring (MANDATORY)** - Every solution, decision, and approach MUST have a confidence score (0.0 → 1.0):
  - **0.9-1.0**: High confidence - Well-tested approach, clear best practices, minimal unknowns, low risk
    - Example: *"Use existing EntityCard pattern - **Confidence: 0.95**"*
  - **0.7-0.8**: Good confidence - Sound approach with minor trade-offs, some uncertainties but manageable
    - Example: *"Extract component to ProfileSidebar/ vs profile/ProfileSidebar/ - **Confidence: 0.8**"*
  - **0.5-0.6**: Moderate confidence - Multiple valid approaches, unclear winner, significant trade-offs (**requires listing alternatives**)
  - **0.0-0.4**: Low confidence - High uncertainty or risk (**MUST escalate or rethink approach**)
  - **Required: Score these elements** - Problem identification, overall solution approach, each major implementation decision, each alternative considered, individual implementation steps
  - **Minimum standard**: Any plan with <3 confidence scores is incomplete
- **Reflect and fix weak solutions**: any solution scoring below 0.8 should be reconsidered, refined, or escalated with clear reasoning about trade-offs.

## Non-Negotiables
- **Do not modify routing**: no edits to `src/app/AppLayout.tsx`, `src/app/SmartRouter.tsx`, or route order/params.
- **API endpoint changes must follow existing conventions**: update endpoints only via `src/services/api/*`, preserve `fetchData` behavior and established patterns.
- **Surgical changes only**: touch as few files as possible, prefer additive changes, minimize code deltas.

## Significant Change Escalation (Required)

### When to Escalate (ASK BEFORE PROCEEDING)
- **Confidence <0.8** on any major decision
- Modifying routing, API patterns, or shared utilities
- Touching >5 files for a single feature
- Changing behavior of existing components
- Any "Non-Negotiable" violation
- Deleting or renaming existing exports

### Escalation Template (MANDATORY FORMAT)
If a change is non-trivial or risks behavior shifts, **ask before proceeding** using this format:
```
**ESCALATION REQUIRED**

Proposed change:
- Files: [list]
- Impact: [what changes]

Cause for concern:
- What could break: [specific risks]
- Affected areas: [screens/features]

Why this is the best option:
- [reasoning with Confidence: X.XX]

Alternatives considered:
1. [Option] - Confidence: X.XX - Why rejected: [reason]
2. [Option] - Confidence: X.XX - Why rejected: [reason]

Risk mitigation:
- [how to reduce risk]
```

## Project Layout Index (src/)
- `src/app/`: feature screens and page-level logic.
  - Each feature has `index.tsx` and optional `data/` hooks (React Query).
  - Examples: `EntityDetail`, `UniversalDetail`, `UserProfile`, `BidDetail`, `ListingDetail`.
- `src/services/api/`: API service layer built on `fetchData`.
- `src/types/`: schema-driven DTOs and shared types.
- `src/components/`: reusable UI components and domain cards.
- `src/components/Tailwind/`: shared design-system components (exported via `index.ts`).
- `src/helpers/`: small utility helpers (e.g., `slugify`).
- `src/constants.ts`: domain config constants (e.g., `brandTagConfig`).
- `src/styles/`: global style overrides.
- `src/index.tsx`: app entry point.

## Routing (Read-Only Reference)
- Routing is defined in `src/app/AppLayout.tsx` and `src/app/SmartRouter.tsx`.
- SmartRouter decides between `/:username/:referenceCode` and `/:brandSlug/:entityName`.
- **Do not change routes, params, route order, or SmartRouter logic**.
- If a new feature appears to require a new route, **escalate** before any changes.

## Data Access Conventions
### Service Layer (`src/services/api/*`)
- All API calls go through `fetchData` in `src/services/api/index.ts`.
- Each resource has a service module: `Entity`, `Listing`, `Bid`, `List`, `Profile`, `User`, `Universal`.
- Service methods are thin wrappers around `fetchData` with resource-specific URLs.
- **Endpoint updates are allowed only in services**, and must preserve:
  - `fetchData` usage
  - URL path conventions (leading `/resource/...`)
  - Existing auth/proxy behavior

### React Query Hooks (`src/app/**/data/*`)
- Hooks are `useFetchX` and live in `data/`.
- Standard pattern:
  - `useQuery` (or `useQueries` for batch).
  - `queryKey` includes the key and identifying params.
  - `queryFn` builds `URLSearchParams` with `include` fields.
  - `enabled` guards on required params.
  - `staleTime` is typically 5 minutes.
  - Returned object exposes `{ data, refetch, isLoading, error }` with feature-specific names.
- Keep hooks read-only by default. Add writes only when explicitly requested.

### Query Parameter Conventions
- Use `URLSearchParams` and repeated `include` keys for relational includes.
- Include chains follow domain needs (e.g., `entity.brand`, `entity.product`, `entity.entityTags.tag`).

## Type Conventions
- DTOs come from `src/types/index.ts` (generated schemas in `oak-api-schemas.ts`).
- Prefer DTO types (`EntityDto`, `ListingDto`, etc.) for component and hook typing.
- Extend payload types by mirroring existing `*Payload` patterns.

## UI and Component Conventions
### Layout
- Use `Layout` (`src/components/Layout.tsx`) for app screens that need header/footer structure.
- Landing page (`src/landing/`) is separate and uses its own layout.

### Tailwind Component Library
- Prefer components from `src/components/Tailwind` for common UI primitives.
- Import from `src/components/Tailwind/index.ts` instead of deep paths when possible.

### Domain Cards
- `EntityCard` is the base renderer for listings/bids/entities.
- `ListingCard` and `BidCard` are thin wrappers around `EntityCard`.

## Styling Conventions
- Global styles live in `src/index.css` and `src/styles/globals.css`.
- Prefer Tailwind classes for component styling.
- Avoid changes to global styles unless required and approved.

## React Best Practices
- **Avoid unnecessary complexity**: Keep components simple. Prefer declarative Tailwind utilities over custom hooks/state when possible.
- **Minimize hook usage**: Avoid overusing `useMemo`, `useEffect`, `useRef`, and `useState` for simple layouts or styling logic.
- **Prefer CSS/Tailwind over JavaScript**: Use responsive Tailwind classes (e.g., `md:grid-cols-3`, `lg:grid-cols-4`) instead of `useEffect` + resize listeners.
- **Performance alignment**: Only optimize when necessary. Avoid premature optimization that adds complexity without measurable benefit.
- **Simplicity first**: If a solution requires multiple hooks, refs, or complex state management, consider if there's a simpler declarative approach.

## Component Reusability & DRY Principles (MANDATORY)

### Strict Rules
- **Never duplicate JSX blocks >15 lines** across feature screens - extract to `src/components/`
- **2+ uses = extract immediately** - Don't wait for 3rd duplication
- **Layout patterns** (sidebars, headers, panels, toolbars) MUST be components, not copy-paste
- **Before creating any feature screen**, check for existing patterns:
  ```bash
  # Search for similar layouts
  rg "aside.*w-full.*lg:w-\[362px\]" src/
  rg "ProfileStatItem" src/
  ```

### Component Extraction Checklist
Before adding JSX to a feature screen:
- [ ] Searched codebase for similar UI patterns
- [ ] Checked if >1 screen will need this pattern
- [ ] If reusable: extracted to `src/components/` with props
- [ ] If domain-specific: used appropriate folder (`entity/`, `list/`, etc.)

### Folder Structure
- **Top-level** `src/components/ComponentName/` - Cross-domain components (ProfileSidebar, DetailLayout)
- **Domain-specific** `src/components/domain/ComponentName/` - Single-domain components (entity/EntityCard)

### Code Review Requirements
Plans must include:
- **DRY Check Results** - What you searched for, what you found
- **Reusability Justification** - Why creating new vs reusing existing

## Images and Assets
- Image URLs commonly use `REACT_APP_S3_IMAGE_BASE_URL`.
- Use fallback handling patterns (e.g., `onError`).

## Environment and Proxy Rules (Reference Only)
- `fetchData` handles proxy, guest token, and base URL logic.
- Do not change proxy/auth logic without escalation.

## Surgical Change Checklist
- Smallest possible file set.
- Prefer additive changes over edits.
- Follow existing naming, hooks, and service patterns.
- Avoid changing routing or shared fetch behavior.

## Required Plan Structure

Every plan MUST include these sections in this order:

### 1. Problem Analysis
- What needs to change and why
- **Confidence: X.XX** - How certain are you about the problem?
- Evidence: specific files, lines, patterns found in codebase

### 2. Proposed Solution
- High-level approach
- **Confidence: X.XX** - How certain this is the best approach?
- Why chosen over alternatives

### 3. Alternatives Considered
- Option 1: [description] - **Confidence: X.XX** - Why not chosen?
- Option 2: [description] - **Confidence: X.XX** - Why not chosen?
- (List at least 2 alternatives for any significant decision)

### 4. Implementation Steps
1. Step description - **Confidence: X.XX** - Risks: [list]
2. Step description - **Confidence: X.XX** - Risks: [list]
(Each step must have confidence score and identified risks)

### 5. Files Modified
- List all files to create or modify
- Mark **NEW** files vs edits
- Estimate lines added/changed per file

### 6. DRY Check Results
- **Searched for**: [patterns, similar implementations]
- **Found existing**: [components/patterns discovered]
- **Decision**: Reusing [what] OR Creating new because [why with confidence score]

### 7. Escalation Check
- **Escalation needed?** YES/NO
- If YES: Use escalation template from "Significant Change Escalation" section
- If NO: Explain why changes are low-risk

### 8. Overall Confidence Score
**X.XX** - Summary of solution confidence with key uncertainties listed

**Minimum Requirements:**
- At least 5 confidence scores throughout the plan
- DRY check must show actual search performed
- Alternatives section required for any confidence <0.9
- Escalation check cannot be skipped
