# ShotBoard — Naming & Output Scan Upgrade Plan

## Context

ShotBoard is a single-file HTML canvas app (`index.html`) for visually organising AI-generated video clips into Shots and Scenes, then batch-renaming and exporting them.

The user's real-world naming convention looks like this:

```
SPT25_SC38_000301_001_V.mp4
│     │    │      │   └── V = video (S = still source image)
│     │    │      └────── 001 = sequence number (within the shot)
│     │    └───────────── 000301 = shot number (zero-padded 6 digits)
│     └────────────────── SC38 = scene identifier
└──────────────────────── SPT25 = project code
```

Sometimes source images have a `temp` flag (e.g. `SPT25_SC38_002temp_S`) meaning "still in progress." This flag must carry through to the video names when present.

There are two problems to solve:

1. **The naming template system is too rigid.** Tokens like `{shot}` are hardcoded to `padStart(3, '0')`. The user needs per-token control over padding width, custom prefixes/suffixes within tokens, and support for flags like `temp`.

2. **No output folder awareness.** If the user exports clips across sessions, the app can overwrite existing files because it always starts numbering from 001. It needs to scan the output folder and continue from the highest existing number.

---

## Part 1: Flexible Naming Template

### Problem

The current `applyTemplate()` function (around line ~900 of `index.html`) does simple string replacement with hardcoded padding:

```js
.replace(/\{shot\}/gi, shot ? String(shot.number).padStart(3, '0') : '000')
.replace(/\{version\}/gi, 'v' + String(version).padStart(2, '0'))
```

This produces `SH001` and `v01`. The user needs `000301` (6-digit pad) and `001` (3-digit pad, no prefix) — and full control over these choices.

### Solution: Token Format Syntax

Replace the simple `{token}` syntax with an extended format:

```
{token:pad:prefix:suffix}
```

All parts after the token name are optional. Examples:

| Template token | Output for value 38 | Notes |
|---|---|---|
| `{scene}` | `38` | No padding (raw value) |
| `{scene:2}` | `38` | Pad to 2 digits |
| `{scene:4}` | `0038` | Pad to 4 digits |
| `{scene:2:SC}` | `SC38` | Pad to 2 + prefix |
| `{shot:6}` | `000301` | Pad to 6 digits |
| `{seq:3}` | `001` | Sequence number, pad to 3 |
| `{version:2:v}` | `v01` | Pad to 2 + prefix |

For backward compatibility, bare `{token}` should still work and default to sensible padding (3 digits), matching current behaviour.

### Implementation

#### 1. New token parser function

```js
function parseToken(tokenStr) {
  // tokenStr is the content between { and }, e.g. "scene:2:SC"
  const parts = tokenStr.split(':');
  return {
    name: parts[0].toLowerCase(),
    pad: parts[1] ? parseInt(parts[1], 10) : 0,
    prefix: parts[2] || '',
    suffix: parts[3] || '',
  };
}
```

#### 2. New token formatter function

```js
function formatToken(value, pad, prefix, suffix) {
  const padded = pad > 0 ? String(value).padStart(pad, '0') : String(value);
  return prefix + padded + suffix;
}
```

#### 3. Rewrite `applyTemplate()` to use regex-based token expansion

Replace the chain of `.replace()` calls with a single regex pass:

```js
function applyTemplate(fileId, seqOffset = 0) {
  // ... (existing file/shot/scene lookup) ...

  const tokenValues = {
    project: { value: state.projectName, isString: true },
    scene:   { value: scene ? scene.name : 'UNSCENED', isString: true },
    shot:    { value: shot ? shot.number : 0 },
    seq:     { value: version + seqOffset },  // sequence within shot
    version: { value: version },
    nn:      { value: seqNum },
    suffix:  { value: t ? t.suffix : '', isString: true },
    prefix:  { value: t ? t.prefix : '', isString: true },
    flag:    { value: t ? t.flag : '', isString: true },  // NEW: for 'temp' etc.
    original:{ value: original, isString: true },
    date:    { value: today(), isString: true },
    ext:     { value: ext.replace('.', ''), isString: true },
  };

  const result = state.template.replace(/\{([^}]+)\}/g, (match, tokenStr) => {
    const { name, pad, prefix, suffix } = parseToken(tokenStr);
    const tokenDef = tokenValues[name];
    if (!tokenDef) return match; // unknown token, leave as-is
    if (tokenDef.isString) return prefix + String(tokenDef.value) + suffix;
    return formatToken(tokenDef.value, pad, prefix, suffix);
  });

  return result + ext;
}
```

#### 4. Add `{flag}` token and `temp` support

Add a `flag` property to each thumbnail in state. When files are imported, parse the source filename to detect `temp`:

```js
// During file ingestion, after creating the thumbnail entry:
const tempMatch = f.name.match(/(\d+)(temp)/i);
if (tempMatch) {
  thumbnail.flag = 'temp';
}
```

The user can then include `{flag}` in their template. If the source image was `SPT25_SC38_002temp_S`, the flag is `temp`. If not, it's empty string — so it vanishes cleanly from the output name.

Example template: `{project}_{scene:2:SC}_{shot:6}{flag}_{seq:3}_V`

- With flag: `SPT25_SC38_000301temp_001_V.mp4`
- Without flag: `SPT25_SC38_000301_001_V.mp4`

#### 5. Update the toolbar UI

The current token chips in the toolbar show clickable badges like `{project}`, `{scene}`, etc. Update these to reflect the new syntax:

- Keep the existing chips but add a small "format help" tooltip or info line below the template input, e.g.:
  `Format: {token:padding:prefix} — e.g. {scene:2:SC} → SC38`
- Add new chips: `{seq}`, `{flag}`
- Rename `{version}` chip description to clarify it's the left-to-right position within a shot
- Add `{seq}` as the "sequence number" (this is what gets offset-aware numbering from Part 2)

#### 6. Default template

Change the default template from:

```
{project}_{scene}_SH{shot}_{version}
```

To:

```
{project}_{scene:2:SC}_{shot:6}_{seq:3}_V
```

This produces `SPT25_SC38_000301_001_V` out of the box.

---

## Part 2: Output Folder Awareness (Overwrite Protection)

### Problem

The user works across multiple sessions. If they export clips for shot `SPT25_SC38_002` in session 1 (producing `_001_V`, `_002_V`, `_003_V`), then generate more clips in session 2, the app starts numbering from 001 again and overwrites existing files.

The current `copyFileToDirectory()` has a reactive collision check (appends `_001` if a file exists), but this produces wrong names like `SPT25_SC38_002_001_V_001.mp4` — a mangled double-suffix.

### Solution: Pre-export scan

Before renaming/exporting, scan the output folder to find the highest existing sequence number for each shot prefix, then start new numbering from `max + 1`.

### Implementation

#### 1. New function: `scanOutputFolder(dirHandle)`

```js
async function scanOutputFolder(dirHandle) {
  // Returns a Map: shotPrefix -> highestSeqNumber
  // e.g. "SPT25_SC38_000301" -> 4
  const maxSeq = new Map();
  
  // Scan all scene subdirectories
  for await (const [name, handle] of dirHandle.entries()) {
    if (handle.kind !== 'directory') continue;
    for await (const [fileName, fileHandle] of handle.entries()) {
      if (fileHandle.kind !== 'file') continue;
      // Extract shot prefix and sequence number from filename
      // Pattern: everything before the last _NNN_V (or _NNN_V.ext)
      const match = fileName.match(/^(.+?)_(\d{3,})_V\b/i);
      if (match) {
        const prefix = match[1];
        const seq = parseInt(match[2], 10);
        const current = maxSeq.get(prefix) || 0;
        if (seq > current) maxSeq.set(prefix, seq);
      }
    }
  }
  
  // Also scan root level (in case files aren't in scene subfolders)
  for await (const [fileName, handle] of dirHandle.entries()) {
    if (handle.kind !== 'file') continue;
    const match = fileName.match(/^(.+?)_(\d{3,})_V\b/i);
    if (match) {
      const prefix = match[1];
      const seq = parseInt(match[2], 10);
      const current = maxSeq.get(prefix) || 0;
      if (seq > current) maxSeq.set(prefix, seq);
    }
  }
  
  return maxSeq;
}
```

**Note on the regex:** The pattern `_(\d{3,})_V` matches the sequence number. This assumes `_V` is always the final token before the extension. If the user changes their template to end differently, this regex needs to be adapted. A more robust approach: store the template's "sequence token position" and derive the regex from the template itself. But for v1, hardcoding `_V` is fine since that's the user's established convention.

#### 2. Integrate into `generatePreview()` and `exportFiles()`

In `exportFiles()`, after the user picks the output directory:

```js
// After: const sceneDirs = { ... };
const existingSeqs = await scanOutputFolder(outDir);
```

Then when building the preview/applying the template, for each file:

```js
// Determine this file's shot prefix (everything before {seq} in the template)
const shotPrefix = buildShotPrefix(fileId); // e.g. "SPT25_SC38_000301"
const offset = existingSeqs.get(shotPrefix) || 0;
// Pass offset to applyTemplate
const newName = applyTemplate(fileId, offset);
```

#### 3. Helper: `buildShotPrefix(fileId)`

This function applies the template but stops before the `{seq}` token, giving you the prefix to match against existing files:

```js
function buildShotPrefix(fileId) {
  // Apply template but replace {seq} with empty, trim trailing separators
  const fullName = applyTemplate(fileId, 0);
  // Remove the sequence portion and everything after it
  // Based on template structure, the prefix is everything before _NNN_V
  const match = fullName.match(/^(.+?)_\d{3,}_V/i);
  return match ? match[1] : fullName;
}
```

#### 4. Update `generatePreview()` to show offset-aware names

The rename preview modal should reflect the actual names that will be written, including offsets. This means `generatePreview()` needs access to the output folder scan results.

Add an optional parameter:

```js
function generatePreview(existingSeqs = null) {
  // ... existing logic ...
  // When computing version/seq for each file:
  const shotPrefix = buildShotPrefix(t.fileId);
  const offset = existingSeqs ? (existingSeqs.get(shotPrefix) || 0) : 0;
  // version = left-to-right position + offset
}
```

#### 5. Show existing file count in preview modal

In the rename preview modal header, add an info line:

```
⚠ Output folder contains existing files. Numbering continues from highest existing sequence.
SPT25_SC38_000301: 3 existing → new files start at 004
```

This gives the user confidence that nothing will be overwritten.

#### 6. Remove the reactive collision suffix from `copyFileToDirectory()`

The current fallback that appends `_001` is now unnecessary (and harmful — it produces mangled names). With the pre-scan in place, collisions should not occur. Keep it as a safety net but log a warning if it triggers, so the user knows something unexpected happened.

---

## Part 3: YES-first / MAYBE-second Ordering

### Problem

Files arrive from the triage tool sorted into YES and MAYBE folders. When they're imported into ShotBoard, this distinction is lost — all files are treated equally.

### Solution

During import, detect whether a file's `relativePath` contains `/yes/` or `/maybe/` (case-insensitive). Store this as a `triage` property on the file object.

In `applyTemplate()`, when computing the left-to-right version order within a shot, sort YES files before MAYBE files (and within each group, maintain left-to-right order).

### Implementation

#### 1. Tag files on import

In the file ingestion logic, after creating the file entry:

```js
const pathLower = f.relativePath.toLowerCase();
if (pathLower.includes('/yes/') || pathLower.includes('\\yes\\')) {
  f.triage = 'yes';
} else if (pathLower.includes('/maybe/') || pathLower.includes('\\maybe\\')) {
  f.triage = 'maybe';
} else {
  f.triage = 'unknown';
}
```

#### 2. Sort within shots

In `applyTemplate()` where it builds `thumbsInShot` sorted by x-position, add a triage-priority pre-sort:

```js
const triagePriority = { yes: 0, unknown: 1, maybe: 2 };
const thumbsInShot = shot.fileIds
  .map(fid => {
    const thumb = state.thumbnails.find(x => x.fileId === fid);
    const file = state.files.find(x => x.id === fid);
    return thumb ? { ...thumb, triage: file ? file.triage : 'unknown' } : null;
  })
  .filter(Boolean)
  .sort((a, b) => {
    const tp = (triagePriority[a.triage] || 1) - (triagePriority[b.triage] || 1);
    if (tp !== 0) return tp;
    return a.x - b.x || a.y - b.y;
  });
```

#### 3. Visual indicator

On thumb cards, show a small coloured dot or badge indicating triage status:
- Green dot for YES
- Amber dot for MAYBE
- No dot for unknown

This helps the user see the ordering logic visually on the canvas.

---

## Build Order

| Step | What | Why first |
|------|------|-----------|
| 1 | Token parser + formatter functions | Foundation for everything else |
| 2 | Rewrite `applyTemplate()` with regex expansion | Enables flexible naming |
| 3 | Add `{flag}` token + temp detection on import | Completes the naming model |
| 4 | Update toolbar chips + default template | UX matches new system |
| 5 | `scanOutputFolder()` function | Safety net for cross-session work |
| 6 | Integrate scan into `exportFiles()` + `generatePreview()` | Makes the safety net active |
| 7 | Show offset info in preview modal | User confidence |
| 8 | Triage tagging on import | Enables YES/MAYBE ordering |
| 9 | Triage-aware sort in `applyTemplate()` | Completes the ordering |
| 10 | Visual triage indicators on thumbnails | Polish |

---

## Test Cases

### Naming Template
- [ ] `{project}_{scene:2:SC}_{shot:6}_{seq:3}_V` → `SPT25_SC38_000301_001_V.mp4`
- [ ] `{shot:3}` → `301` (3-digit pad)
- [ ] `{shot:6}` → `000301` (6-digit pad)
- [ ] `{shot}` → `301` (no pad = raw value, backward compat)
- [ ] `{scene:2:SC}` → `SC38` (prefix applied)
- [ ] `{flag}` with temp source → `temp`; without → empty string (no stray underscores)
- [ ] Unknown token `{foo}` → left as literal `{foo}`
- [ ] Template with no tokens → static filename (collision expected, shown in preview)

### Output Folder Scan
- [ ] Empty output folder → seq starts at 001
- [ ] Output folder has `_001_V`, `_002_V`, `_003_V` → new files start at `_004_V`
- [ ] Output folder has files from different shots → each shot gets correct offset
- [ ] Output folder has files in scene subfolders → all scanned correctly
- [ ] Output folder has unrelated files → ignored (no regex match)
- [ ] Mixed: some shots have existing files, some don't → offsets applied per-shot

### YES/MAYBE Ordering
- [ ] Import from YES folder → files tagged `triage: 'yes'`
- [ ] Import from MAYBE folder → files tagged `triage: 'maybe'`
- [ ] Import from unknown folder → files tagged `triage: 'unknown'`
- [ ] Within a shot: YES files get lower seq numbers than MAYBE files
- [ ] Within same triage tier: left-to-right canvas order preserved
- [ ] Triage badges visible on thumbnails

### Edge Cases
- [ ] Template produces identical names for two files → collision shown in preview, export blocked
- [ ] User changes template after preview → re-preview shows updated names
- [ ] `{flag}` adjacent to `_` in template doesn't produce double underscore when flag is empty
- [ ] Very large sequence numbers (>999) handled gracefully
- [ ] Session save/load preserves triage tags and flag values
