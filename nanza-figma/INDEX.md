---
tags: [nanza-figma, index]
---

# nanza-figma

Figma → code handoffs. A designer edits a frame in Figma. The `figma-handoff` skill in
nanza-mobile reads that frame over the Figma MCP and writes **one implementation prompt per
frame** into `handoffs/`. A coding session in nanza-mobile then implements from that file.

These prompts live here, not in the app repo, so the repo stays code-only and every handoff
stays visible in Obsidian.

- **How a handoff is made:** `nanza-mobile/.claude/skills/figma-handoff/SKILL.md`. Run it with
  `/figma-handoff`, or paste "give me a prompt for: <link> / i changed: … / do not change: …".
- **How the app side uses one:** [[../nanza-mobile/solution-designs/figma-handoff|Figma handoff (mobile)]].
- **The code → Figma direction** (token sync, screen capture) is the `figma-sync` skill and the
  `nanza-figma-sync` repo.

## Lifecycle

`open` → `in-progress` (a session is implementing) → `built` (code written, uncommitted) →
`done` (QA'd and committed). Change the `status:` frontmatter and add a line to the file's Log
at each step. Answers to a handoff's open questions go **into the handoff file**, so the
implementing session sees them.

## Handoffs

| Date | Handoff | Screen | Status |
|------|---------|--------|--------|
| 2026-10-23 | [[handoffs/2026-10-23-makeoffer-listing-card\|Make Offer — listing card]] | MakeOffer | built |
