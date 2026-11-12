---
tags: [nanza-api, nanza-mobile, plan]
status: plan — decided 2026-11-06 (voice); API + mobile built 2026-11-06, uncommitted, not deployed, no device QA
---

# FaBrary export (collection CSV + deck import link)

Let a user push a Nanza collection (binder) into [FaBrary](https://fabrary.net) two ways:

1. **FaBrary collection** → a CSV in FaBrary's collection-import format (one row per
   printing, `Have` = our quantity).
2. **FaBrary deck** → a `https://fabrary.net/decks?tab=import&…` link the user copies and
   opens; FaBrary fills the import form, the user presses Import.

Both start from the collection detail share button.

## UX (nanza-mobile)

Collection detail (`CollectionDetailScreen`) — the `share-3` dark-circle button over the
banner stops calling `guardedShare` directly and opens a **bottom sheet** with three rows:

| Row | Action |
|---|---|
| **Share** | today's behaviour: `guardedShare({ message: buildShareUrl(referenceCode) })` |
| **FaBrary collection** | `GET /list/:id/export/fabrary-collection` → CSV file → native share sheet (file, `text/csv`, name `<collection>-fabrary.csv`) |
| **FaBrary deck** | opens a **second sheet** (form-bar style, same chips as listing / bid forms): **Name** text field prefilled with `list.displayName ?? list.name`, editable, max 50; **Format** chip row — Blitz, Classic Constructed, Draft, Golden Age, Living Legend, Open, Sealed, Silver Age, Ultimate Pit Fight (none preselected; FaBrary defaults by hero). **Generate link** → `GET /list/:id/export/fabrary-deck?format=&name=` → `{ url }` → shows the link with a **Copy link** button (`Clipboard.setString`), toast "Copied". No auto-forward to the browser. |

Text labels only for now — no FaBrary logo in the sheet (Skylar may add one later).
Rows visible to the same viewers that see share today (`referenceCode && canShare`).

## API (nanza-api)

Both exports are **server-side** — the mobile collection grid is paged (21/page) and
doesn't hold the whole list, and the field mapping needs `product` + `set` + `entityTags`
for every item. Pure read, no schema change.

### `GET /list/:id/export/fabrary-collection`

Response `text/csv`, `Content-Disposition: attachment; filename="<name>-fabrary.csv"`.
Header exactly as FaBrary's template:

```
Identifier,Name,Pitch,Set,Set number,Edition,Foiling,Treatment,Have,Want in trade,Want to buy,Extra for trade,Extra to sell
```

One row per `EntityList` row (per printing). Only `Have` is filled; the four want/extra
columns are blank.

### `GET /list/:id/export/fabrary-deck?format=<Format>&name=<name>`

Response `{ url }`. Builds
`https://fabrary.net/decks?tab=import&format=<encoded>&name=<encoded ≤50>&cards=A,B,C…`
where `cards` is the **base set number** (suffixes and `U-`/`R-` stripped) repeated
`quantity` times, comma-joined. `format` optional; validated against the nine formats.
`name` optional. Set identifiers (`OMN143`) are what FaBrary documents for printings;
no slug needed here.

### Field mapping (collection CSV)

Source of truth for the parse is `parseProductNumber` in `src/utils/tcgApiMatch.ts`
(grammar `[U-|R-] <base> [-<suffix>]`). New pure helper `src/utils/fabrary.ts` reuses it.

| FaBrary column | Nanza source | Notes |
|---|---|---|
| `Identifier` | `slug(entity.displayName) + '-' + slug(pitch)` | e.g. `a-bit-off-the-side-red`. Slug: lowercase, strip apostrophes/commas/punctuation (`10,000 Year Reunion` → `10000-year-reunion`, `A Moment's Peace` → `a-moments-peace`), non-alphanumerics → `-`, collapse/trim dashes. No pitch → no suffix (`Identifier` = name slug). |
| `Name` | `entity.displayName` | |
| `Pitch` | `color` brand-tag value on the entity (`Red` / `Yellow` / `Blue`) | Blank for pitch-less cards (heroes, equipment). |
| `Set` | `set.displayName` | Our set display names already mirror FaBrary's ("Part the Mistveil", "Arcane Rising"); **Promos** → `Promos`. Reconcile against FaBrary's catalog CSV on Desktop before QA. |
| `Set number` | `parsed.base` of `product.number` | `U-ARC123-RF` → `ARC123`. |
| `Edition` | from `product.number` prefix | `U-` → `Unlimited`. `R-` → `First`. No prefix → `First` **if the same base exists elsewhere in our catalogue with `U-`** (i.e. the set has an Unlimited run), else blank. Alpha not emitted. |
| `Foiling` | suffix | `RF` → `Rainbow`, `CF` → `Cold`, `GF` → `Gold`, none → blank. |
| `Treatment` | suffix | `MV` → `Full Art`, `EA` → `Extended Art`, `AA` → `Alternate Art`, else blank. (FaBrary also has Alternate Text / Alternate Border and comma-joined combos; not produced by us.) |
| `Have` | `EntityList.quantity ?? 1` | |

### Edition — known catalogue gap (not this task)

Our catalogue stores Unlimited as `U-<base>` but has **no marker for First edition**: the
plain `<base>` means "First" when a `U-` sibling exists, and "no edition" otherwise
(e.g. `MPW075`). The export infers it (one `SELECT DISTINCT base` over products with `U-`
in the same brand). Fixing the data model (e.g. always writing `R-` for first edition, as
`parseProductNumber` already accepts) is a separate decision Skylar will take with his
partner.

## Built 2026-11-06 (uncommitted)

- nanza-api: `src/utils/fabrary.ts` (mappers, smoke-tested against FaBrary catalog rows), two GET routes in `src/routers/list.ts` with a shared `loadFabraryCards` loader. Typecheck + lint clean.
- nanza-mobile: `fetchText` in `services/api/index.ts`, `listService.exportFabraryCollectionCsv` / `exportFabraryDeck`, `ActionModal` share sheet + `FabraryDeckSheet` in `CollectionDetailScreen`. Typecheck + lint clean.
- **Android gap:** RN's `Share.share` can't share a file on Android, so Android gets the CSV as text. Fix later with a share-file library if FaBrary's import needs a file.
- Pitch tag: matched by tag name `color` / `colour` / `pitch` (case-insensitive) — confirm the FAB brand tag is named `color` on dev.

## Files to touch

nanza-api
- `src/utils/fabrary.ts` — new: `slugIdentifier`, `mapRow`, `buildDeckUrl`, `FABRARY_FORMATS`.
- `src/routers/list.ts` — two GET routes above (same viewer check as `/list/:id/entities`).
- `src/services/` — only if a shared "load full list with product/set/color tag" loader
  doesn't exist; else inline Prisma in the router.

nanza-mobile
- `src/screens/collections/CollectionDetailScreen/index.tsx` — share → sheet.
- new `ShareCollectionSheet` + `FabraryDeckSheet` under that screen folder.
- `src/services/api/List.ts` — `exportFabraryCollection(id)` (blob → file), `exportFabraryDeck(id, params)`.
- i18n copy: Share · FaBrary collection · FaBrary deck · Name · Format · Generate link · Copy link · Copied.

## Reference

- FaBrary deck-import URL doc (pasted 2026-11-06): `cards` = card or set identifiers, one per copy,
  comma-separated; `format` optional; `name` ≤ 50; hero included by identifier.
- FaBrary catalog CSV (17,565 rows, header + vocabulary): `~/Desktop/Blank FaBrary collection November 2, 2026.csv`.
  Vocabulary observed — Pitch: Red/Blue/Yellow; Edition: First/Unlimited/Alpha; Foiling: Rainbow/Cold/Gold;
  Treatment: Extended Art / Full Art / Alternate Art / Alternate Text / Alternate Border (+ comma combos).
