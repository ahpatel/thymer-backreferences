# 2026-02-23: Backreferences Sort Controls (v1)

Goal: Add sortable backreferences with per-page saved preferences in local storage. Ship v1 with requested sort options and document persistence tradeoffs.

## Plan

- [x] Confirm SDK capabilities + plugin architecture constraints and finalize v1 scope with user
- [x] Record implementation plan before coding
- [x] Implement sort state model (per-panel + per-page persistence in localStorage)
- [x] Implement sort UI controls (sort menu + Asc/Desc) in footer header
- [x] Implement sorting comparators for Property References and Linked References groups
- [x] Add per-page preference sync across open panels for the same record
- [x] Update docs with sorting behavior + persistence tradeoffs + future options
- [x] Verify behavior (manual checklist + syntax check)
- [x] Commit completed subtasks incrementally

## Review

- Added v1 sort menu with options: Page Last Edited, Reference Activity, Reference Count, Page Title, Page Created Date.
- Added Ascending/Descending controls and live re-ordering without re-fetching.
- Persisted sort preferences per record GUID in localStorage, with cross-panel sync for panels showing the same page.
- Documented persistence tradeoff (local per-device) and deferred workspace-sync/default-sort roadmap.
- Verification:
  - `node -e "const fs=require('fs'); const src=fs.readFileSync('plugin.js','utf8'); new Function(src);"` (syntax parse)
  - Node sanity check with stubbed `AppPlugin` for sort helper methods.
  - Manual Thymer UI checks still required for final visual/interaction confirmation.
