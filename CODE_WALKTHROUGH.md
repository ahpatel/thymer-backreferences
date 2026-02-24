# Backreferences Plugin - Code Walkthrough

This is a guided tour of `plugin.js` for the Backreferences plugin, explained in Thymer's Plugin SDK context.

Scope:
- File: `plugin.js`
- Plugin type: Global app plugin (`class Plugin extends AppPlugin`)
- Goal: For every editor panel that is showing a record, mount a footer UI that shows:
  - Property References: records that point to the current record via record-link properties
  - Linked References: line items found via Thymer search (`@linkto = "<recordGuid>"`)

If you want to follow along, open `plugin.js` and jump between the section headers. The code is already organized with big comment banners:
- `// ---------- Panel lifecycle ----------`
- `// ---------- Mounting ----------`
- `// ---------- Click handling ----------`
- `// ---------- Refresh orchestration ----------`
- `// ---------- Event-driven freshness ----------`
- `// ---------- Grouping + rendering ----------`
- `// ---------- CSS ----------`

---

## 1) Thymer Context: What APIs Exist

When this plugin runs inside Thymer, you get a few important objects from the SDK:

- `this.ui`: UI helpers (command palette commands, inject CSS, create/switch panels, toasters, etc.)
- `this.events`: event bus (panel navigation/focus/close, record/line updates, reload)
- `this.data`: data access (records, users, and search)
- `panel`: a UI panel object. In particular we use:
  - `panel.getId()`
  - `panel.getActiveRecord()`
  - `panel.getElement()` (DOM)
  - `panel.navigateTo({ type: 'edit_panel', rootId, subId, workspaceGuid })`

Also note the packaging constraint:
- Thymer strips top-level code outside the `Plugin` class (see the comment at the top of `onLoad()`), so everything must be reachable from methods.

---

## 2) High-Level Architecture (Mental Model)

The plugin is basically three pipelines that cooperate:

1) Panel tracking
   - Listen for panel navigation/focus/close.
   - For each panel showing a record, ensure a footer DOM node is mounted.

2) Refresh pipeline
   - Debounce refresh calls (avoid spamming search/property scans while the UI is churny).
   - For each refresh, load linked references + property references concurrently.
   - Render results into the footer.

3) Click delegation
   - One click handler on the footer root.
   - Individual elements (buttons/spans) declare an action via `data-action="..."`.
   - The click handler decides what to do (open record, open line, toggle collapse, etc.).

---

## 3) Boot + Shutdown

### `onLoad()`

Key responsibilities:

1) Initialize state:
   - `_panelStates` is a `Map<panelId, state>` so we can handle multiple open panels.
   - `_collapsed` and `_propGroupCollapsed` are loaded from `localStorage`.

2) Register UI surface area:
   - Command palette command: "Backreferences: Refresh (Active Page)"

3) Subscribe to events:
   - `panel.navigated` / `panel.focused`: mount/refresh when the active record changes.
   - `panel.closed`: remove state + DOM.
   - `reload`: refresh everything.
   - `lineitem.updated` / `lineitem.deleted` / `record.updated`: keep the footer fresh when the underlying graph changes.

4) Kick the initial render:
   - Grab `this.ui.getActivePanel()` and run `handlePanelChanged(panel, 'initial')`.

Why this shape?
- In Thymer, panels are the primary unit of navigation. If you show the same record in two panels, they should each get their own footer and refresh cycles.

### `onUnload()`

Cleanup responsibilities:
- Unsubscribe from all events (`this.events.off(id)` for stored handler IDs).
- Remove the command palette command.
- Dispose every panel state (disconnect observers, remove DOM nodes, clear timers).

This keeps the plugin reload-safe (which matters while iterating in the plugin editor).

---

## 4) Panel Lifecycle + Per-Panel State

### `handlePanelChanged(panel, reason)`

This is the entrypoint any time a panel navigates or gets focus.

What it does:
- Computes `panelId` via `panel.getId()`.
- Reads the panel's active record and its `guid`.
- If there is no active record, we dispose the footer for that panel (nothing to show).
- Otherwise:
  - get/create a `state` object for that panel
  - mount the footer (DOM)
  - schedule a refresh

Important detail:
- It determines whether the record changed (`recordChanged`) and uses that to decide whether to refresh immediately (`force: recordChanged`).
  - Navigation should refresh immediately.
  - Focus can be debounced if you just clicked back into the panel.

### `getOrCreatePanelState(panel)`

State stored per panel:
- `recordGuid`: which record the panel is currently showing
- `rootEl`, `bodyEl`, `countEl`: cached DOM handles
- `observer`: a `MutationObserver` to remount when Thymer re-renders the panel
- `refreshTimer`: debounce timer handle
- `refreshSeq`: monotonically increasing counter used to ignore stale async refreshes
- `searchOpen`, `searchQuery`: per-panel footer search UI state
- `sortBy`, `sortDir`: per-panel active sort mode + direction
- `sortMenuOpen`: whether the sort menu popover is open
- `lastResults`: cached unfiltered results so typing does not re-fetch data
- `isLoading`: toggles a loading CSS class (minor affordances)

Why store DOM handles?
- Rendering is imperative (we mutate the footer DOM). Keeping references avoids repeated `querySelector` calls and reduces "did my node disappear" bugs.

### `disposePanelState(panelId)`

Disposal steps:
- clear any pending refresh timer
- disconnect the mutation observer
- remove the root DOM node
- delete state from `_panelStates`

---

## 5) Mounting: Getting the Footer Into Thymer's DOM

### `mountFooter(panel, state)`

Thymer's editor can re-render parts of a panel, which can drop plugin-owned DOM nodes.

So this function:

1) Gets `panelEl = panel.getElement()`.
2) Finds a reasonable mount point in that panel:
   - `.page-content` OR `.editor-wrapper` OR `#editor` OR the panel itself
3) (Re)builds the footer DOM if needed:
   - if `state.rootEl` is missing or no longer connected, build again
4) Ensures the root is attached to the correct container.
5) Installs a `MutationObserver` once per panel:
   - if Thymer churns the DOM and our node disappears, we remount on the next tick.

Why MutationObserver?
- Without it, a footer can "randomly" vanish when the host app re-renders.
- Using a next-tick remount (`setTimeout(..., 0)`) avoids fighting with Thymer's own DOM updates.

### `buildFooterRoot(state)`

Builds a DOM subtree like:

```text
div.tlr-footer
  div.tlr-header
    button[data-action=toggle]
    div.tlr-title
    div.tlr-count (data-role=count)
    div.tlr-spacer
    button[data-action=toggle-sort-menu] (sort icon)
    div.tlr-sort-menu (sort-by options + Asc/Desc)
    div.tlr-search-wrap (input + clear; hidden unless open)
    button[data-action=toggle-search] (search icon; hidden when open)
  div.tlr-body (data-role=body)
```

Notable design choices:
- We attach *one* click handler to `root` and use `data-action` for delegation.
- We render interactive rows as `<button>` where appropriate.
  - This gives you keyboard focus/activation behavior "for free" and makes it easier to treat rows as clickable targets.

---

## 6) Click Delegation + Navigation

### `handleFooterClick(e)`

This is the single click handler for everything inside the footer.

Pattern:
1) Find the nearest clicked element with `data-action`:
   - `e.target.closest('[data-action]')`
2) Use `data-action` to select behavior.

Supported actions:
- `toggle`: collapse/expand the entire footer across *all* panels
- `toggle-prop-group`: collapse/expand a property group (persisted in local storage)
- `toggle-search`: open/close the footer search input
- `toggle-sort-menu`: open/close the sort popover
- `set-sort-by`: switch sort mode
- `set-sort-dir`: switch Asc/Desc direction
- `clear-search`: clear query (or close search if already empty)
- `open-record`: open a record (used by record headers and property reference record rows)
- `open-line`: open a record with `subId = lineGuid` (best-effort focus)
- `open-ref`: open a referenced record from inside the segment renderer

Why this approach?
- It keeps DOM creation simple: elements just declare intent (`data-action`, `data-record-guid`, etc.).
- You avoid wiring up dozens/hundreds of per-row listeners.

### `openRecord(panel, recordGuid, subId, e)`

This wraps Thymer navigation:
- Uses `panel.navigateTo({ type: 'edit_panel', rootId: recordGuid, subId, workspaceGuid })`.
- Supports "open in new panel" if Ctrl/Cmd is held:
  - `this.ui.createPanel({ afterPanel: panel })` then navigate the new panel.

Why pass `subId`?
- Thymer uses `subId` as a "secondary focus" target.
- For line references, we pass the line item's guid so the editor can try to focus that line.
  - This is best-effort: if the host app changes the meaning of `subId`, we'd need to adjust.

---

## 7) Local UI State (Collapsed + Group Collapsed + Sort Preferences)

### `_collapsed`

- Stored in local storage via `saveCollapsedSetting()`.
- On load, we attempt to read the new key, then migrate from an older key (`backlinks`) if present.
- If nothing is stored, we fall back to plugin config (`custom.collapsedByDefault`).

### `_propGroupCollapsed`

- Stored as a JSON array of property names.
- `isPropGroupCollapsed(propName)` and `setPropGroupCollapsed(propName, collapsed)` wrap the `Set`.

### `_sortByRecord`

- Stored as a JSON object in localStorage keyed by `recordGuid`.
- Value shape: `{ sortBy, sortDir }`.
- Defaults to `Page Last Edited` + `Descending` when no saved value exists.
- When one panel changes sort for record X, all open panels currently showing record X update immediately.

Why localStorage?
- This is a UI preference, not workspace data.
- It should be fast and per-user, and not require data model changes.

---

## 8) Refresh Orchestration (Debounce + Stale-Result Guard)

### `scheduleRefreshForPanel(panel, { force, reason })`

- Clears any existing `state.refreshTimer`.
- Schedules `refreshPanel(panelId)` after either:
  - `0ms` if `force` (record changed)
  - `_refreshDebounceMs` otherwise

Why debounce?
- `panel.focused`, `lineitem.updated`, and general UI churn can create bursts of refresh triggers.
- Debounce reduces redundant work and keeps the app snappy.

### `refreshPanel(panelId, { reason })`

This is where the data work happens.

Key steps:
1) Re-read the active recordGuid from the panel (don't trust cached state).
2) Ensure the footer is mounted.
3) Increment `refreshSeq` and capture the current sequence number.
4) Mark loading state.
5) Read config:
   - `maxResults`
   - `showSelf`
6) Start two async operations *in parallel* using `Promise.allSettled`:
   - Search: `this.data.searchByQuery(@linkto = "<guid>", maxResults)`
   - Property scan: `getPropertyBacklinkGroups(...)
7) Ignore stale results:
   - If `refreshSeq` changed while awaiting, return early.
8) Normalize errors and feed into `renderReferences(...)`.

Why `Promise.allSettled` instead of `Promise.all`?
- If search fails but property scan succeeds (or vice versa), we still want to render partial results.

Why the `refreshSeq` guard?
- A slow refresh could finish after a newer one.
- Without the guard, you'd render out-of-date results (classic async race).

---

## 9) Event-Driven Freshness

These handlers decide when to refresh without requiring the user to click "Refresh":

### `handleRecordUpdated(ev)`

- Property references come from record properties, not line items.
- So when record properties change, refresh all panels (debounced).

### `handleLineItemUpdated(ev)`

- Only refresh panels that might be affected.
- We inspect the updated line's segments; if it references record X, and panel P is currently showing X, schedule refresh for P.

This is a big performance win compared to refreshing every footer for every line edit.

### `handleLineItemDeleted()`

- We don't know what was referenced, so we refresh all (debounced). Deletes are rare enough to justify it.

---

## 10) Data Sources: Property References vs Linked References

### Linked References: `@linkto` search

- Query: `@linkto = "<recordGuid>"`
- API: `this.data.searchByQuery(query, maxResults)`
- Output we care about: `result.lines` (line items that matched)

### Property References: scan all records

Thymer's linked reference search does not include record-link properties, so we do our own scan:

`getPropertyBacklinkGroups(_targetRecord, targetGuid, { showSelf })`:

1) Load all records: `this.data.getAllRecords()`
2) For each record:
   - skip if it is the target and `showSelf` is false
   - inspect `record.getAllProperties()`
   - if a property references the targetGuid, bucket the source record under that property name
3) Convert the map into groups and sort them.

Potential caveat:
- This is O(number_of_records * number_of_properties).
- It's acceptable for small/medium workspaces, but could be optimized with caching/indexing if needed.

### Property value parsing helpers

`getPropertyCandidateValues(prop)` tries multiple accessors:
- `prop.text()` (common)
- `prop.choice()` (fallback)

Then `expandPossibleListString(...)` tries to interpret multi-values:
- JSON array string: `["guid1","guid2"]`
- comma separated: `guid1, guid2`
- newline separated

This is defensive coding for how record-link properties may serialize.

---

## 11) Grouping + Rendering

### Linked references grouping

`groupBacklinkLines(lines, targetGuid, { showSelf })`:

- Dedupes lines by `line.guid`.
- Groups by source record guid.
- Keeps lines grouped by source record; final page ordering is applied later in render based on the current sort mode.
- Sorts lines within each group by `createdAt` asc (read in-order).

### `renderReferences(state, ...)`

This is the top-level renderer:
- Clears the body.
- Computes counts and writes the header count string.
- Applies the selected record sort across both sections using shared metrics:
  - `Page Last Edited`
  - `Reference Activity`
  - `Reference Count`
  - `Page Title`
  - `Page Created Date`
- Renders "Property References" section:
  - error OR empty OR grouped list
- Renders divider + "Linked References" section:
  - error OR grouped list

Why render from scratch?
- The datasets are not huge and it avoids tricky incremental diffing.
- It keeps correctness simple: your DOM always matches your current data snapshot.

### Property references UI

`appendPropertyReferenceGroups(container, groups)`:
- For each property name:
  - Renders a collapsible header (`data-action=toggle-prop-group`) with an arrow caret.
  - Renders each referencing record as a button (`data-action=open-record`).

### Linked references UI

`appendLinkedReferenceGroups(container, groups, { maxResults })`:
- For each source record:
  - Renders a record header button (`data-action=open-record`).
  - Renders each matching line as a row button (`data-action=open-line`) with `data-line-guid`.
  - Line content is rendered via `appendSegments`.

---

## 12) Segment Rendering (Why the Backlinks Look Like Thymer)

Line items in Thymer are stored as "segments" (typed pieces of text).

`appendSegments(container, segments)` handles a subset:
- `text`: append a text node
- `bold` / `italic` / `code`: wrap in styled spans
- `link` / `linkobj`: render as `<a>` with `target=_blank`
- `hashtag`, `datetime`, `mention`: render as styled spans
- `ref`: render as a clickable span that opens the referenced record (`data-action=open-ref`)

Important design decision:
- For `ref` we render the title as plain text (no `[[...]]`).
  - Double brackets are a Roam/Obsidian convention, not a native Thymer convention.

---

## 13) CSS + Theme Integration

`injectCss()` uses `this.ui.injectCSS(...)` to attach all styling.

Theme alignment strategy:
- Use Thymer/theme CSS variables when possible.
- For links:
  - `--ed-link-color`, `--link-color`
  - `--ed-link-hover-color`, `--link-hover-color`

Why the `--ed-*` vars?
- Example themes in the Thymer knowledgebase define both `--link-*` and `--ed-link-*`.
- Using them makes backlinks match the editor's link styling more closely.

Borders in light themes:
- Divider and header borders are `2px` to keep them visible on bright backgrounds.
- Border color is still theme-driven via `--border-subtle` (with an rgba fallback).

---

## 14) Known Limitations / Future Improvements

- Native editor "blink" highlight on refs is not exposed by the public SDK.
  - We can approximate attention with `this.ui.bounce(element)` for plugin DOM, but it is not the same effect.
- Property reference scanning can be expensive in very large workspaces.
  - A future improvement could cache a reverse index of record-link relationships.
- Mount-point selection (`findMountContainer`) is best-effort and depends on host DOM structure.
- Sort preferences are currently local per browser profile/device (localStorage).
  - A future option could sync these via workspace plugin config for cross-device/shared behavior.

---

## 15) Practical Debugging Tips

- Use command palette: `Backreferences: Refresh (Active Page)`.
- Use the footer search icon to filter/highlight results without re-fetching.
- If the footer disappears, it should remount automatically (MutationObserver). If it does not, inspect the panel DOM and update `findMountContainer()`.
- To debug data:
  - temporarily add `console.log(...)` inside `refreshPanel` and `renderReferences`.
