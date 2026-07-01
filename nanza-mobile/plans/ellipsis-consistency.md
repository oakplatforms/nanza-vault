# Ellipsis ("more") icon consistency

## Problem (reported by user)
The three-dot "more" ellipsis renders inconsistently across screens:
- Profile (own) vs user-profile: one looks **lighter** than the other.
- Messages thread header: ellipsis is **clearly bigger** than elsewhere.

## Root cause
The glyph is `<Icon name="more" />` → `assets/icons/more.svg`, whose viewBox is **16×3**
(a wide, short horizontal ellipsis). It is sized everywhere with numbers tuned for
*square* icons, and via three different rendering paths with different prop conventions,
sizes, and default colors.

### Inventory of every `more` render

| Where | Size prop | Effective W×H | Color |
|---|---|---|---|
| Header branch A (line 305) — own ProfileScreen | `size={md-1}` = 19 | 19×19 (square box, glyph scaled) | default `secondary[50]` |
| Header branch B (line 433) — collapsing header | `size` | square | default `secondary[300]` ← **lighter, different default** |
| UserProfileScreen rightIcons | `size={md-1}` = 19 | 19×19 | inherits branch default |
| Messages ConversationThread (line 150) | `size={md}` = 20 | **20×20** ← **biggest** | `text` |
| EntityCardBase (line 136) | `width={md-min}` = 18 | 18×24 | `secondary[800]` (dark, for light card) |
| EntityModal (line 641) | `width={md-min}` = 18 | 18×24 | `secondary[800]` |
| FeedBidCard (line 258) | `width={md-xxs}` = **16** | 16×24 | `secondary[50]` |
| FeedBulkCard (line 244) | `width={md-min}` = 18 | 18×24 | `secondary[50]` |
| FeedListingCard (line 338) | `width={md-min}` = 18 | 18×24 | `secondary[50]` |

### Two distinct defects
1. **Size inconsistency**: widths range 16→20 and the prop is sometimes `size`
   (square box) and sometimes `width` (height defaults to 24). Because the glyph
   is 16:3, the box's *width* is what sets the visual dot size — so the messages
   one at 20 reads visibly bigger; the bid card at 16 reads smaller.
2. **Lighter/darker**: the two Header branches default the rightIcons color
   differently (`secondary[50]` vs `secondary[300]`), so own-profile vs
   user-profile ellipses differ in weight even with identical size.

## Proposed fix (size + weight consistency)
Standardize the *width* of every `more` ellipsis to one token so the visual dot
size matches everywhere, and make color explicit per surface (don't rely on the
Header's branch-dependent default).

Recommendation: pick a single width — `theme.icons.md - theme.spacing.min` (18) —
which is already the most common value (4 of the card usages + closest to the
profile headers). Apply it to:
- Messages: change `size={md}` (20) → `width={md - min}` (18).
- Profile headers: pass an explicit `size`/width = 18 AND an explicit `color`
  so the two header branches stop diverging. (rightIcons takes `size`; for a
  16:3 glyph `size` makes it square — acceptable visually but inconsistent with
  the card path which uses width+height 18×24. Cleanest: special-case `more`.)

### Cleanest option — single source of truth
Because `more` is the only non-square icon being sized by these call sites,
the most robust fix is to **normalize `more` inside the `Icon` component**: when
`name === 'more'`, derive height from width using the 16:3 ratio (or just fix
height) so every call site that passes a width/size lands on the same visual size
regardless of which prop it used. Then all call sites only need a consistent
*width/size number* + explicit color.

Color is intentionally surface-dependent (dark dots on light cards =
`secondary[800]`; light dots on dark feed = `secondary[50]`). We will NOT flatten
color to one value — only fix the accidental light/dark divergence between the
two header branches and unify SIZE/WEIGHT as the user asked.

## IMPLEMENTED (target width = 18, user approved)
1. `Icon/index.tsx` — for `name === 'more'`, derive height from width via the 16:3
   ratio so any width/size value yields a consistent ellipsis (no more square-box
   stretch difference between the `size` and `width` call paths).
2. Every `more` call site set to width **18** (`theme.icons.md - theme.spacing.min`):
   - Messages ConversationThread: `size={md}` (20) → `width={md - min}` (18)
   - FeedBidCard: `width={md - xxs}` (16) → `width={md - min}` (18)
   - Header `rightIcons` triggers (`size: theme.icons.md` → `md - min`):
     BulkItems, CreateBid, ScanItemEdit, GroupDetail, CreateListing,
     CollectionDetail, UserCollectionDetail
   - UserProfileScreen: `size: md - 1` → `md - min` (18) + **explicit color**
     `secondary[50]` so it no longer rides the differing Header-branch default
     (this fixes the own-profile-vs-user-profile "lighter/darker" mismatch).
3. Colors left intentionally surface-specific: `secondary[800]` (dark dots on
   light cards: EntityCard/EntityModal), `secondary[50]` (light dots on dark
   surfaces: feed cards + profile header), `text` (dark headers). Only the
   accidental header-branch divergence was removed.

Result: every ellipsis now renders at the same visual width/weight regardless of
screen or render path.
