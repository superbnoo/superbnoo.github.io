# Sheet Editor — View Mode & Print Design Spec

Adds a read-only view mode and print-to-PDF capability to the existing sheet editor.

## Overview

The sheet editor currently has a single edit mode. This spec adds two features:

1. **View mode** — a toggle on the same page that strips away all edit affordances and shows the sheet as a clean, read-only rendering with cell borders, beat labels, chord labels displayed as static text, and the song title centered at the top.
2. **Print to PDF** — a button (visible in view mode) that renders the sheet into a hidden iframe with a print-specific stylesheet and calls `window.print()` from that iframe, producing clean paper output without any site chrome.

## Constraints

- All changes are within the existing `sheet-editor.html` file.
- No new dependencies.
- View mode reuses the existing DOM and render path — toggled via a CSS class on `<body>`, not a separate renderer.
- Print uses a hidden iframe with a self-contained HTML document for full isolation from the main page's styles.

## View Mode

### Toggle mechanism

- In **edit mode**: a "View" button sits in the toolbar's action bar (alongside Add Line, Export, Import). Clicking it switches to view mode.
- In **view mode**: the toolbar hides. A separate `<div id="mode-bar">` appears above the sheet with an "Edit" button (switches back) and a "Print" button.
- Toggling adds or removes a `view-mode` class on `<body>`.
- A JS variable `viewMode` (boolean) tracks the current state.
- When toggling to view mode, any open popup is closed and any selection is cleared.

### What changes visually in view mode

| Element | Edit mode | View mode |
|---|---|---|
| Toolbar panel | Visible (title input, add line, export/import) | Hidden |
| Mode toggle button | Shows "View" label | Shows "Edit" label |
| Print button | Hidden | Visible |
| Header description `<p>` | Visible | Hidden |
| Song title | Editable input in toolbar | Centered heading above the sheet |
| Chord labels | Editable `<input>` elements | Static text (read-only) |
| Cell hover | `--a-tint` background + pointer cursor | No hover effect, default cursor |
| Drag handles | Visible on selected cells | Hidden |
| Cell click | Opens popup (empty) or selects (note) | No interaction |
| "Remove" buttons on lines | Visible | Hidden |
| Beat labels | Visible | Visible |
| Cell borders | Visible | Visible |
| Measure dividers | Visible | Visible |

### View mode title display

When in view mode, a `<div id="view-title">` is shown above the sheet area. It displays `state.title` (or "Untitled" if empty) as a centered heading, styled with `--cond` font, large size, matching the site's `header h1` style but centered.

### CSS approach

All view-mode overrides use the `.view-mode` ancestor selector:

```css
.view-mode #toolbar { display: none; }
.view-mode header p { display: none; }
.view-mode #view-title { display: block; }
.view-mode .cell { cursor: default; }
.view-mode .cell:hover { background: var(--surface-2); }
.view-mode .cell.has-note:hover { background: var(--surface); }
.view-mode .drag-handle { display: none; }
.view-mode .line-foot { display: none; }
.view-mode .popup { display: none !important; }
.view-mode .chord-input { pointer-events: none; border-bottom-color: transparent; }
```

### Mode bar

When in view mode, a small floating bar appears at the top (outside the sheet) with:
- "Edit" button (to switch back)
- "Print" button

This bar is a `<div id="mode-bar">` that is hidden in edit mode and shown in view mode.

## Print via Hidden Iframe

### Trigger

The "Print" button in the mode bar calls a `printSheet()` function.

### Process

1. Create a hidden `<iframe>` (or reuse an existing one): `position: absolute; width: 0; height: 0; border: none;`.
2. Build a self-contained HTML string:
   - `<link>` to Google Fonts for IBM Plex Mono.
   - Inline `<style>` with print-specific rules: white background, black text, clean grid borders, no shadows, proper page margins, `@page { margin: 15mm; }`.
   - Song title as a centered `<h1>`.
   - For each line: chord labels as centered text divs, treble note row as a 32-column CSS grid, beat labels row, bass note row, beat labels row. All static — no inputs, no interactive elements.
3. Write the HTML into the iframe: `iframe.contentDocument.open(); iframe.contentDocument.write(html); iframe.contentDocument.close();`.
4. Wait for fonts to load (`iframe.contentWindow.document.fonts.ready`), then call `iframe.contentWindow.print()`.

### Print stylesheet specifics

- Always light: white background, `#131B24` text (the `--ink` light-mode value).
- Font: IBM Plex Mono for notes and beat labels, IBM Plex Sans for title and chords.
- Grid uses the same 32-column CSS Grid as the editor, with thin solid borders.
- Measure boundaries: thicker left border (matching edit mode).
- No hover, no selection, no drag handles, no remove buttons.
- `@page { margin: 15mm; }` for reasonable paper margins.
- `page-break-inside: avoid` on each line to prevent splitting a line across pages.

### The `buildPrintHTML()` function

Reads from `state` and produces a complete HTML document string. Reuses the same `noteText()`, `getFlat()`, `findSpanOwnerFlat()` logic but generates HTML strings rather than DOM nodes.

## Out of Scope

- Saving the PDF server-side (browser-native "Save as PDF" only).
- Custom page size or orientation controls.
- Header/footer on printed pages.
- Separate view-mode URL or routing.
