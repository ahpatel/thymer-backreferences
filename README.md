# Backlinks (Linked References)

Adds a Roam-style **Linked References** section to the bottom of each record/page.

How it works:

- Uses the Thymer search index via `data.searchByQuery()` with `@linkto = "<recordGuid>"`.
- Renders matching line items grouped by the source record.

## Setup

1. In Thymer, open Command Palette (Ctrl/Cmd+P) and select `Plugins`
2. Create (or open) a **Global Plugin**
3. Paste `plugins/backlinks/plugin.json` into Configuration
4. Paste `plugins/backlinks/plugin.js` into Custom Code
5. Save

Note: This plugin injects its own CSS at runtime; `plugins/backlinks/plugin.css` can stay empty.

## Usage

- Scroll to the bottom of a page to see **Linked References**.
- Click a record header to open the source record.
- Click a reference line to open the source record and (best-effort) focus that line.
- Hold Ctrl/Cmd while clicking to open in a new panel.

Command palette:

- `Backlinks: Refresh (Active Page)`

## Configuration

Edit `custom` in `plugins/backlinks/plugin.json`:

- `maxResults` (number): cap for search results (SDK default is 100; plugin defaults to 200)
- `collapsedByDefault` (boolean): start the footer collapsed
- `showSelf` (boolean): include references from the current record (default false)

## Verification Checklist

1. Open a record A that you know is referenced by other records.
2. Confirm the footer appears at the bottom with a "Linked References" header.
3. Click "Refresh" and confirm results render and are grouped by source record.
4. Click a source record header and confirm it navigates to that record.
5. Ctrl/Cmd-click a source record header and confirm it opens in a new panel.
