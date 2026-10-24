---
tags: [nanza-mobile, solution-design, design-system, migration]
---

# Semantic Token Migration — raw palette scale refs → semantic tokens

## Status: PENDING (deferred from the Aug 2026 token audit)

This completes the rule stated in [[design-system|Design System]]: **components (and style
buckets) reference *semantic* tokens only — never raw palette scales.** A large body of
pre-existing code still reaches into the raw ramps (`theme.colors.secondary?.['400']`,
`primary['300']`, …) instead of the semantic alias that resolves to the same value
(`textSubtle`, `highlight`, …).

**Why it's a separate pass, not part of the Aug-2026 token audit:** these swaps are **P2 /
cosmetic** — the resolved value is identical, so nothing renders differently and no bug is
fixed. They are also **pervasive** (see inventory) and spread across files unrelated to any one
change. Bundling them into an unrelated PR violates the standing rule *"don't retrofit semantic
tokens in unrelated code,"* and doing them piecemeal leaves files half-migrated (worse than
untouched). So they are batched here to be done **consistently, as their own dedicated pass**.

The Aug-2026 audit fixed the real correctness items (hardcoded hex, hex fallbacks, off-scale
sizes, off-scale font sizes). This migration is the remaining P2 tail.

## Inventory (occurrences across `src/`, at time of writing)

| Scale ref | Occurrences | Files |
|---|---|---|
| `secondary['50']` | 52 | 16 |
| `secondary['200']` | 27 | 9 |
| `secondary['300']` | 49 | 22 |
| `secondary['400']` | 58 | 21 |
| `secondary['500']` | 85 | 25 |
| `secondary['600']` | 36 | 17 |
| `secondary['700']` | 56 | 20 |
| `secondary['800']` | 20 | 12 |
| `secondary['950']` | 70 | 20 |
| `primary['300']` | 7 | — |
| `primary['400']` | 14 | — |
| `neutral['950']` | 1 | — |

~**475 occurrences**. Regenerate with:
```
for V in 50 200 300 400 500 600 700 800 950; do
  printf "secondary[%s]: " "$V"; grep -rEn "secondary(\?\.)?\['$V'\]" src --include='*.ts' --include='*.tsx' | wc -l
done
```

## Mapping (value-verified against `tokens/dark.json`)

Prefer the semantic token on the right. **Fallback chains keep the same target** (drop the raw
lead, keep/normalize the tail).

| Raw scale ref | → Semantic token | Notes |
|---|---|---|
| `secondary['50']` | `onSurface` | |
| `secondary['200']` | `cardBackground` | ⚠ verify role — often appears as a *text* color where cardBackground reads wrong |
| `secondary['300']` | `secondaryText` | ✅ clean, single match |
| `secondary['400']` | `textSubtle` | ⚠ value also = `onWell`; for text use `textSubtle` |
| `secondary['500']` | `textDisabled` | ⚠ placeholder-role sites may intend `textPlaceholder` despite value |
| `secondary['600']` | `textPlaceholder` | ✅ clean, single match |
| `secondary['700']` | `textMuted` | ⚠ value also = `textOnSurface` |
| `secondary['800']` | `inactive` | |
| `secondary['950']` | **AMBIGUOUS** — `surface` (fill) / `textAccent` (text) / `inputBackground` | decide per site by role; never blind-swap |
| `primary['300']` | `highlight` | |
| `primary['400']` | `action` | |
| `neutral['950']` | `background` | |

## The one hard rule: role-verify every swap

**Value match ≠ role match.** A token that shares a hex is not automatically the right *name*.
Before swapping, look at the property and intent:

- **Text / icon color** (`color:`, `fill=`, `placeholderTextColor=`) → use the text-role token
  (`secondaryText`, `textSubtle`, `textMuted`, `textPlaceholder`, `textDisabled`).
- **Fill / background** → use the surface-role token (`surface`, `cardBackground`, `onSurface`,
  `background`, `ink`).
- **Border** → `border` / `inactive`.
- Where several semantic tokens share a value (`secondary['950']` especially), **pick by role or
  leave the scale ref** — a wrong-role name is worse than a raw ref.

### Skip (do NOT swap)
- **Fallback tails where the semantic token already leads the chain** — e.g.
  `textSubtle || secondary?.['400'] || border`: the tail never renders, swapping it is noise.
- Any occurrence whose role doesn't match the mapped token.
- Effects/shadows/gradients — out of scope; those are their own (also-deferred) tokenization.

## Suggested execution

Do it as a **dedicated, consistent pass** — ideally one focused PR (or a few by area), not
sprinkled into feature work:

1. Pick a scale ref (start with the ✅ clean ones: `['300']`→`secondaryText`,
   `['600']`→`textPlaceholder`).
2. `grep -rEn "secondary(\?\.)?\['300'\]" src` → for each hit, confirm the property/role.
3. Swap same-role hits; leave/flag the rest. Keep fallback tails' target the same.
4. Save the ambiguous `['950']` for last, deciding each by role.
5. `npx eslint <files>` + `npx tsc --noEmit` must be clean before commit.
6. Keep diffs value-only — no logic, structure, or unrelated edits (same discipline as the
   Aug-2026 audit).

## Provenance

Seeded by the Aug-2026 token audit. The audit's representative anchor list (a *sample*, not the
full set) lived in the repo's gitignored `.plans/token-audit-findings-2026-08-01.md` (§4/§5).
This doc supersedes it as the durable, complete record — regenerate counts from `src` rather
than trusting any static list.
