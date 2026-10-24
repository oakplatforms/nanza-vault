---
tags: [nanza-mobile, nanza-figma, solution-design, design-system, figma]
---

# Figma handoff: implementing a designer's Figma edit

**Status:** set up 2026-10-23 (Skylar, voice). The skill and the first handoff are written; the
pipeline hasn't been run end to end on a second frame yet.

## The loop

1. **Code → Figma.** The `figma-sync` skill pushes tokens and captured screens into Figma.
   The designer works on `· edit me` clones there.
2. **Designer → handoff.** The designer pastes a frame link plus "i changed / do not change".
   `/figma-handoff`, in a nanza-mobile session, reads the frame over the Figma MCP. It
   measures the frame, maps it to `src/`, and writes
   `nanza-vault/nanza-figma/handoffs/<date>-<screen>-<element>.md`.
3. **Handoff → code.** A fresh nanza-mobile session implements from that file (below).

Handoffs live in the vault ([[../../nanza-figma/INDEX|nanza-figma]]), never in this repo. The repo
holds code only; the vault keeps every handoff visible and searchable in Obsidian.

## Implementing a handoff

- Start the session with: *"Implement `nanza-vault/nanza-figma/handoffs/<file>.md`"*, or paste
  its **Prompt** block.
- **Read the Open questions first.** Answers are written inline in the handoff file. An
  unanswered question gets asked during planning. Never guess at it.
- The usual house rules apply: plan with confidence scores, proportional sizing (fractions of the
  402 reference / 362 content width via `thumbSizes.ts`), no inline styles, theme tokens only,
  `/ui-review` before finishing.
- **Shared components get opt-in props.** A handoff is scoped to one frame. If a component it
  touches has other consumers, the change must not leak to them unless the handoff says so.
- **Code stays canonical.** Figma values are proposals. A Figma font or colour that doesn't exist
  in code (e.g. "Balto Mobile") is flagged, not added.
- When done: set the handoff's `status:` to `built`, add a Log line, and update the row in
  `nanza-figma/INDEX.md`. It becomes `done` after QA and commit.

## Why it's shaped this way

- **One file per frame** keeps the designer's request (verbatim), the measurements, the open
  questions and the prompt together, so the implementer and the designer read the same thing.
- **Measured, not eyeballed.** `get_metadata` gives absolute geometry, so gaps and fractions go
  into the prompt as numbers. The implementing session doesn't round-trip to Figma to find them.
- **Traps become questions.** Most edits silently contradict an earlier dated decision (stepper
  hidden at 1, the hugging price box). The skill surfaces those for a human instead of
  overwriting them.
