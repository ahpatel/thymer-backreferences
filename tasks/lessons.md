# Lessons (Backreferences Plugin)

Add entries here after user corrections so repeated mistakes are avoided.

## 2026-02-23

- If users ask for sort options, confirm both naming clarity and directional semantics (Asc/Desc should still read naturally).
- "Most Recent" can be ambiguous in backlink UX; clarify whether recency means page-level edits or reference-line activity.
- For multiplayer plugins, document persistence scope explicitly (local-only vs workspace-shared) before implementation.
- If users request v1 constraints, implement only agreed scope and document deferred roadmap items separately.
- After persistence tradeoff decisions, add explicit docs notes for future toggles (workspace sync, default sort) to preserve intent.
- When users report "had to save in a weird order," assume install-doc ambiguity first: explicitly state that paste order does not matter and that both tabs should be updated before a single save.
- Do not assume a `ti-*` icon name exists; for critical controls, use deterministic plugin-owned glyphs or visible text fallback to avoid blank buttons.
- For header controls that toggle expanded UI (search menus), design DOM order/CSS so controls do not swap sides or push metadata text around.
