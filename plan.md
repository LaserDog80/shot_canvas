# ShotBoard — Implementation Plan

## Goal

Build a single-file HTML application that provides a visual canvas where users drag in video files, arrange thumbnails spatially, group them into Shots and Scenes via box-drawing, then batch-rename and export the files based on their canvas placement.

---

## Requirements

### Core (must have — Phase 1)

1. **Single HTML file** with all CSS and JS inline; no external dependencies except Google Fonts (Inter + JetBrains Mono)
2. **File ingestion** via drag-and-drop (folder scanning) and folder picker button — reuse ShotTriage's `ingestFiles()`, `readEntry()`, `readFilesFromHandle()`, `isVideoFile()`
3. **Thumbnail extraction** from video files (first frame by default, 30% frame on-demand) as small JPEG blobs (max 320px wide)
4. **Infinite canvas** using CSS transforms (`scale()` + `translate()`) on a DOM container — NOT `<canvas>` element
5. **Pan**: Space+drag, middle-mouse drag, two-finger trackpad
6. **Zoom**: Ctrl/Cmd+scroll, pinch gesture. Range 25%–400%
7. **Thumbnail rendering** on canvas: absolutely-positioned DOM elements showing extracted frame + filename in mono text; auto-arranged in grid on import
8. **Thumbnail interaction**: click to select, shift+click multi-select, drag to move, Delete/Backspace to remove from session
9. **Marquee selection**: click-drag on empty canvas draws selection rectangle, selects all intersecting items
10. **Shot creation**: select thumbnails → Ctrl+G or right-click "Create Shot" → named box wraps them with auto-incremented shot number (default start 10, increment 10)
11. **Scene creation**: select shots → Ctrl+Shift+G or right-click "Create Scene" → named box wraps them with user-provided scene name
12. **Drag between groups**: drag thumbnails into/out of shots; drag shots into/out of scenes; parent boxes move children
13. **Naming template system** with tokens: `{project}`, `{scene}`, `{shot}`, `{version}`, `{suffix}`, `{prefix}`, `{ext}`, `{original}`, `{date}`, `{nn}`
14. **Rename preview modal** showing original → new filename mapping, grouped by scene, with collision detection (red highlights, export disabled until resolved)
15. **Export**: copy (never move) files into `[output root]/[SCENE_NAME]/` subfolders with new names, using File System Access API's `showDirectoryPicker()` — reuse ShotTriage's `copyFileToDirectory()`
16. **Toolbar**: project name input, template input, shot numbering controls, thumb size slider, Import/Save/Load/Detect Dupes/Preview Rename/Export buttons
17. **Status bar**: file count, scene count, shot count, ungrouped count, zoom level

### Polish (Phase 2)

18. **Context menu** (right-click): context-appropriate actions on thumbnails, shots, scenes, empty canvas
19. **Inline video playback**: double-click thumbnail → lightweight `<video>` overlay; click away to close
20. **Per-thumbnail suffix/prefix**: click thumbnail → small panel to add/edit suffix text
21. **Export collision detection** at preview stage + import collision detection (identical filenames from different source folders)
22. **Duplicate detection**: file-size-based clustering (reuse+extend ShotTriage's `flagDuplicates()`) with badge on thumbnails
23. **Session save/load**: export/import `.shotboard.json` with canvas state, positions, groups, template config; re-link files by `relativePath` on load
24. **Undo**: state snapshots for grouping/ungrouping/move actions (Ctrl+Z)
25. **Thumbnail size slider**: adjustable thumb size in toolbar
26. **Keyboard shortcuts**: full set (Ctrl+G, Ctrl+Shift+G, Delete, Ctrl+A, Ctrl+Z, Ctrl+S, Space+drag, Ctrl+scroll, Escape, F2)
27. **Inline editing**: double-click shot/scene label to rename in-place

### Stretch (Phase 3 — if time allows)

28. Canvas minimap
29. Snap-to-grid option
30. Auto-arrange within shots
31. Batch suffix/prefix application to selected thumbnails

---

## Technical Approach

### File: `index.html` (single file, ~2000–2500 lines)

The file is organized into three inline blocks:

```
<!DOCTYPE html>
<html>
<head>
  <!-- Google Fonts links -->
  <style>
    /* 1. CSS Design System (~350 lines) */
    /* 2. Canvas-specific styles (~150 lines) */
    /* 3. Component styles (toolbar, status bar, modal, context menu) (~150 lines) */
  </style>
</head>
<body>
  <!-- 4. HTML structure (~100 lines) -->
  <script>
    /* 5. State & constants (~60 lines) */
    /* 6. Utility functions (~50 lines) — reused from ShotTriage */
    /* 7. File ingestion (~200 lines) — reused from ShotTriage */
    /* 8. Thumbnail extraction (~60 lines) */
    /* 9. Canvas engine (~250 lines) — pan, zoom, coordinate transforms */
    /* 10. Selection system (~150 lines) — click, shift-click, marquee */
    /* 11. Drag & drop on canvas (~200 lines) — move thumbnails, reparent */
    /* 12. Grouping (~200 lines) — shot/scene create, dissolve, reparent */
    /* 13. Rendering (~300 lines) — render canvas items, update positions */
    /* 14. Naming template (~100 lines) — token expansion, preview generation */
    /* 15. Rename preview modal (~150 lines) */
    /* 16. Export (~100 lines) — copy files with new names */
    /* 17. Session save/load (~100 lines) */
    /* 18. Context menu (~80 lines) */
    /* 19. Undo system (~60 lines) */
    /* 20. Event wiring & init (~150 lines) */
  </script>
</body>
</html>
```

**Estimated total: ~2,500 lines**

---

### Architecture Detail

#### State Object

```javascript
const state = {
  projectName: 'PROJECT',
  files: [],          // [{id, file, name, relativePath, objectUrl, size, thumbnailUrl, possibleDupe}]
  nextId: 0,
  thumbnails: [],     // [{fileId, x, y}]
  shots: [],          // [{id, name, number, suffix, fileIds:[], x, y, w, h}]
  scenes: [],         // [{id, name, shotIds:[], x, y, w, h, outputDir}]
  nextShotId: 0,
  nextSceneId: 0,
  panX: 0, panY: 0, zoom: 1,
  template: '{project}_{scene}_SH{shot}_{version}',
  shotIncrement: 10, shotStartNumber: 10,
  thumbSize: 160,
  selectedItems: [],  // [{type, id}]
  dragState: null,
  mode: 'select',
  outputDirHandle: null,
  undoStack: [],
};
```

#### CSS Strategy

- Reuse the ShotTriage design system variables verbatim (dark theme, Inter/JetBrains Mono, `--bg-*`, `--border-*`, `--text-*`, `--accent`, etc.)
- Reuse button (`.btn`, `.btn-accent`), modal (`.modal-overlay`, `.modal-box`), and progress bar styles
- New canvas-specific styles:
  - `#canvas-viewport` — `overflow: hidden; position: relative;` fills remaining viewport below toolbar
  - `#canvas-world` — `transform-origin: 0 0; transform: scale(zoom) translate(panX, panY);` — the transformed container
  - `.thumb-card` — absolutely positioned, shows JPEG thumbnail + filename label
  - `.shot-box` — absolutely positioned, thin solid border, shot number badge
  - `.scene-box` — absolutely positioned, dashed border, scene name label, subtle tint background
  - `.selected` — blue accent border highlight
  - `.marquee` — selection rectangle with semi-transparent fill
  - `#context-menu` — fixed-position dark menu

#### HTML Structure

```html
<div id="app">
  <!-- Toolbar (sticky top) -->
  <header id="toolbar">
    <!-- Logo, Import, Save, Load buttons -->
    <!-- Template input, Project name input, Shot numbering controls -->
    <!-- Thumb size slider -->
    <!-- Detect Dupes, Preview Rename, Export buttons -->
  </header>

  <!-- Drop zone (shown before files loaded) -->
  <div id="drop-zone">...</div>

  <!-- Canvas viewport (shown after files loaded) -->
  <div id="canvas-viewport">
    <div id="canvas-world">
      <!-- Dynamically inserted: .scene-box, .shot-box, .thumb-card -->
    </div>
    <div id="marquee" class="hidden"></div>
  </div>

  <!-- Drop overlay (for adding more files) -->
  <div id="drop-overlay" class="hidden">...</div>

  <!-- Rename preview modal -->
  <div id="rename-modal" class="modal-overlay hidden">...</div>

  <!-- Context menu -->
  <div id="context-menu" class="hidden">...</div>

  <!-- Status bar (sticky bottom) -->
  <footer id="status-bar">...</footer>
</div>
```

#### Key Function Groups

**Canvas Engine** — Handles coordinate transforms between screen space and canvas world space:
- `screenToWorld(screenX, screenY)` — convert mouse position to canvas coordinates
- `worldToScreen(worldX, worldY)` — convert canvas position to screen position
- `applyTransform()` — update `#canvas-world` CSS transform from state
- `onWheel(e)` — zoom toward cursor position
- `onPanStart/Move/End(e)` — space+drag or middle-mouse panning
- `clampZoom(z)` — enforce 0.25–4.0 range

**Selection System:**
- `selectItem(type, id, additive)` — click/shift-click select
- `deselectAll()` — clear selection
- `startMarquee(x, y)` / `updateMarquee(x, y)` / `endMarquee()` — rectangle selection
- `getItemsInRect(x, y, w, h)` — hit test for marquee

**Grouping:**
- `createShot(fileIds)` — wrap thumbnails in a shot box, prompt for number
- `createScene(shotIds)` — wrap shots in a scene box, prompt for name
- `dissolveShot(shotId)` — ungroup thumbnails, remove shot box
- `dissolveScene(sceneId)` — ungroup shots, remove scene box
- `addToShot(fileId, shotId)` / `removeFromShot(fileId, shotId)`
- `addToScene(shotId, sceneId)` / `removeFromScene(shotId, sceneId)`
- `autoShotNumber()` — next available number in sequence

**Rendering:**
- `renderCanvas()` — clear and rebuild all DOM elements in `#canvas-world`
- `renderThumbnail(thumb)` — create/update `.thumb-card` element
- `renderShot(shot)` — create/update `.shot-box` element with children
- `renderScene(scene)` — create/update `.scene-box` element with children
- `updateStatusBar()` — refresh counts in status bar
- Uses `requestAnimationFrame` for smooth pan/zoom updates

**Naming:**
- `applyTemplate(file, shot, scene)` — expand all tokens, return new filename
- `generatePreview()` — build array of `{original, newName, scene, shot, version, collision}` for all grouped files
- `detectExportCollisions(preview)` — find duplicate output names
- `renderPreviewModal(preview)` — populate and show the modal table

**Export:**
- `exportFiles()` — main flow: prompt for output dir if needed, create scene subfolders, copy files with progress
- Reuse `copyFileToDirectory()` from ShotTriage

**Thumbnail Extraction:**
- `extractThumbnail(file, seekRatio)` — create temporary `<video>`, seek to ratio, draw to offscreen `<canvas>`, return JPEG blob URL (max 320px wide)
- `extractFirstFrame(file)` → calls `extractThumbnail(file, 0)` — used at import
- On hover/toggle: `extractThumbnail(file, 0.3)` for differentiation frame

**Session:**
- `saveSession()` — serialize state (minus File objects and blob URLs) to JSON, trigger download as `.shotboard.json`
- `loadSession(json)` — reconstruct canvas layout, show missing-file placeholders, await re-import to re-link by `relativePath`

**Undo:**
- `pushUndo(description)` — deep-clone relevant state slices onto stack
- `undo()` — pop and restore
- Captures: thumbnail positions, shots, scenes, selectedItems

---

### Build Order (Phase 1 — MVP)

Each step produces a testable increment:

| Step | What | Depends On | Approx Lines |
|------|------|-----------|-------------|
| 1 | HTML skeleton + CSS design system | — | 500 |
| 2 | File ingestion (drag-drop + folder picker) | 1 | 200 |
| 3 | Thumbnail extraction + auto-arrange on canvas | 2 | 100 |
| 4 | Canvas pan/zoom engine | 1 | 200 |
| 5 | Thumbnail click-select + drag-move | 3, 4 | 200 |
| 6 | Marquee selection | 5 | 100 |
| 7 | Shot creation (Ctrl+G) + shot box rendering | 5, 6 | 200 |
| 8 | Scene creation (Ctrl+Shift+G) + scene box rendering | 7 | 150 |
| 9 | Drag between groups (reparenting) | 7, 8 | 150 |
| 10 | Naming template system + toolbar controls | 1 | 100 |
| 11 | Rename preview modal with collision detection | 10 | 200 |
| 12 | Export (copy files to scene subfolders) | 11 | 100 |
| 13 | Status bar | 7, 8 | 50 |

**Phase 1 subtotal: ~2,050 lines → testable MVP**

Phase 2 adds ~450 lines: context menu, inline playback, suffix editing, duplicate detection, session save/load, undo, keyboard shortcuts, thumb size slider.

---

## Test Cases

### File Ingestion
- [ ] Drop a folder containing video files → thumbnails appear auto-arranged on canvas
- [ ] Drop a second folder → new thumbnails added without losing existing ones
- [ ] Drop non-video files → silently ignored
- [ ] Drop nested folder structure → all video files found recursively
- [ ] Use "Import Folder" button → same result as drag-drop
- [ ] Files with identical names from different folders → both import, tooltip shows full path

### Canvas Navigation
- [ ] Space + click-drag → canvas pans smoothly
- [ ] Ctrl/Cmd + scroll → canvas zooms toward cursor
- [ ] Zoom clamps at 25% minimum and 400% maximum
- [ ] Zoom level shown in status bar updates in real-time
- [ ] Pinch gesture on trackpad → zoom works

### Selection & Movement
- [ ] Click thumbnail → selected (blue border)
- [ ] Click empty space → deselect all
- [ ] Shift+click → add to selection
- [ ] Drag selected thumbnail(s) → move on canvas
- [ ] Marquee drag on empty space → selection rectangle appears
- [ ] Marquee intersects thumbnails → they become selected
- [ ] Delete/Backspace with selection → removes from session (not from disk)
- [ ] Ctrl+A → selects all thumbnails

### Shot Grouping
- [ ] Select 3 thumbnails → Ctrl+G → shot box appears with default number (010)
- [ ] Shot box shows number badge in top-left
- [ ] Second shot auto-numbers to 020
- [ ] Drag a thumbnail into an existing shot → it joins that shot
- [ ] Drag a thumbnail out of a shot → it becomes ungrouped
- [ ] Double-click shot label → edit number inline
- [ ] Right-click shot → context menu with Rename, Dissolve options
- [ ] Dissolve shot → thumbnails become ungrouped, box removed

### Scene Grouping
- [ ] Select 2 shots → Ctrl+Shift+G → scene box appears, prompts for name
- [ ] Scene box shows name as large label
- [ ] Drag shot into/out of scene works
- [ ] Moving scene box moves all child shots and their thumbnails
- [ ] Dissolve scene → shots become ungrouped, scene box removed

### Naming Template
- [ ] Default template `{project}_{scene}_SH{shot}_{version}` renders correctly
- [ ] Changing project name updates preview
- [ ] `{shot}` renders as 3-digit zero-padded number
- [ ] `{version}` sequences v01, v02, v03 within each shot
- [ ] `{date}` renders as YYYYMMDD
- [ ] `{original}` preserves original filename
- [ ] Extension is always preserved from original file
- [ ] Custom template with arbitrary text between tokens works

### Rename Preview
- [ ] "Preview Rename" opens modal with table grouped by scene
- [ ] Original → New filename shown for every grouped file
- [ ] Ungrouped thumbnails listed with warning "will NOT be exported"
- [ ] Export collisions highlighted in red
- [ ] Export button disabled when collisions exist
- [ ] Closing modal returns to canvas

### Export
- [ ] Export prompts for output directory if none selected
- [ ] Creates scene-name subfolders in output directory
- [ ] Files copied (not moved) with new names
- [ ] Progress bar shows during copy
- [ ] Original files untouched after export
- [ ] Collision avoidance appends `_001` suffix if file exists in output
- [ ] Summary shown on completion

### Session Save/Load
- [ ] "Save Session" downloads `.shotboard.json`
- [ ] JSON contains canvas positions, shots, scenes, template, project name
- [ ] JSON does NOT contain video file data
- [ ] "Load Session" → file picker → reconstructs canvas layout
- [ ] After load, re-importing source folders re-links files by relativePath
- [ ] Missing files shown as greyed-out placeholders
- [ ] New unmatched files appear as ungrouped thumbnails

### Performance
- [ ] 50+ files load without freezing
- [ ] Pan/zoom stays smooth with 50+ thumbnails
- [ ] Memory stays reasonable (thumbnails are small JPEG blobs, not `<video>` elements)
- [ ] Object URLs revoked when files removed from session

---

## Decisions (resolved)

1. **Version ordering within shots**: Left-to-right canvas position within the shot box. Thumbnails further left get lower version numbers (v01, v02, ...).

2. **Shot box auto-sizing**: Yes — boxes auto-resize to fit their contents. No manual resize handle needed.

3. **Scene output directory**: Scene name as a subfolder of the main output directory. No custom per-scene paths. (`[output root]/[SCENE_NAME]/`)

4. **Template input UX**: Small row of clickable token chips below the template input (e.g., `{project}` `{scene}` `{shot}` `{version}` ...) so the user can see what's available and click to insert. Simple, not autocomplete.

5. **Thumbnail hover (30% frame)**: Deferred to Phase 2. Phase 1 uses first-frame thumbnails only.

6. **Undo granularity**: Structural changes only — create/dissolve shot, create/dissolve scene, add/remove from group, delete from session. Individual thumbnail drags are NOT undo-able (keeps the undo stack clean and performance smooth).
