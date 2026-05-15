# Ak@sh — In-Canvas Document Editor Spec

**Status:** Design locked, pre-implementation
**Target:** Ak@sh v2 — extends the existing single-file canvas workspace into a desktop-installable infinite-canvas document workbench.

---

## 1. Goals

Add the ability to **open documents directly inside the canvas as floating, resizable, searchable editor windows**, so a user can have multiple files visible at once, zoom across them, and edit them in place.

Concretely:

- A workspace has a **pinned tray** of files and folders.
- The user drags a pinned file onto the canvas, which creates a **docwindow** at that position with a default size — resizable thereafter.
- Each docwindow renders a live editor or viewer appropriate to the file kind: Markdown, source code, PDF, XLSX, DOCX.
- Multiple docwindows coexist on a canvas at different positions. Canvas zoom and pan let the user see all of them at once or zoom into one.
- Each docwindow has an **inner zoom** (independent of canvas zoom) so deep zoom is comfortable.
- Each docwindow has its own **in-document search**.
- **Global search** spans canvas items *and* document contents across all workspaces.
- The app is **installable on macOS, Linux, and Windows** with a small, signed installer.

## 2. Non-goals (initially)

- Real-time multiplayer collaboration.
- Round-trip .docx editing (read-only via Mammoth is fine to start).
- Mobile/tablet support (Tauri 2 supports it, but UX work is out of scope).
- Custom syntax-highlighting themes per file (defer; one good theme is enough).
- Built-in version history beyond a single migration backup.

## 3. Locked architecture decisions

| # | Decision | Rationale |
|---|---|---|
| D1 | One canvas per workspace now; schema accommodates many. | Lets us add a canvas-switcher UI later without a data migration. Always address canvases by id (`canvases/<id>.canvas.json`), never special-case `main`. |
| D2 | **Per-canvas** docwindow placement; **workspace-shared** viewstate (cursor, scroll, current page, current sheet). | Window position is a layout property of the canvas; cursor/page is a property of the file. Same `notes/x.md` opened on canvas A and canvas B sits at different spots but resumes at the same cursor line. |
| D3 | Autosave on debounce (~500ms after last keystroke). `Cmd/Ctrl+S` forces an immediate flush + triggers search-index refresh for that file. | Matches Notion / Obsidian / Figma expectations. The force-save lets users push a doc into global search instantly. |
| D4 | Move to Tauri 2 for distribution; keep current HTML/CSS/JS frontend. | Tauri uses the OS native WebView (~3 MB bundles, ~30–50 MB RAM idle vs Electron's ~96 MB / 150–300 MB) and `tauri-action` produces signed installers for mac/win/linux from one workflow. The web frontend is unchanged. |
| D5 | Custom canvas (current code), not tldraw / Excalidraw. | Existing canvas already has pan, zoom, frames, selection, lasso, snap, multi-select, undo, tagging, search, multi-canvas. tldraw is paid for production and would be a near-total rewrite. |
| D6 | **CodeMirror 6** for all text/code editing surfaces (.md, source code). | ~300 KB core vs Monaco's 5–10 MB; per-instance language config (Monaco has global state that collides with multiple editors); built-in `@codemirror/search` covers in-doc search; clean extension API. |

## 4. Tech stack

| Concern | Tool | License |
|---|---|---|
| Desktop shell | **Tauri 2** | Apache-2.0 / MIT |
| Bundler / dev server | Vite | MIT |
| Markdown / code editor | **CodeMirror 6** + lang packs | MIT |
| Markdown rendering (preview) | `markdown-it` or `marked` | MIT |
| PDF viewer | **Mozilla PDF.js** | Apache-2.0 |
| XLSX (read) | **SheetJS** (`xlsx`) | Apache-2.0 |
| XLSX (edit, optional) | **Univer** | Apache-2.0 — verify per use |
| DOCX (read) | **Mammoth.js** (→ HTML) | BSD-2-Clause |
| Cross-workspace index | IndexedDB (web) / SQLite via Tauri plugin (desktop) | — |

Reasoning notes are in §3; check each upstream license before shipping.

## 5. Folder layouts

### 5.1 On-disk workspace (what the user sees)

```
my-workspace/
├── .akash/                          ← all Ak@sh metadata
│   ├── workspace.json
│   ├── canvases/
│   │   └── c_main.canvas.json
│   ├── viewstate/
│   │   └── <pathhash>.json
│   ├── thumbnails/
│   │   └── <itemId>.webp
│   ├── assets/                      ← internal blobs (pasted images)
│   │   └── <uuid>.png
│   ├── index/
│   │   └── content.json             ← per-workspace search index
│   └── backups/
│       └── pre-v2.json              ← migration safety net
└── (user files anywhere they like)
    ├── notes/ideas.md
    ├── docs/spec.pdf
    └── data/budget.xlsx
```

**Rule:** Ak@sh never copies user files. It references them by path relative to the workspace root. Only `.akash/` is owned by the app.

### 5.2 Repo source layout

```
akash/
├── .github/workflows/release.yml    ← tauri-action multi-OS build
├── src/                             ← frontend
│   ├── index.html
│   ├── main.ts
│   ├── canvas/
│   │   ├── world.ts                 ← viewport, pan, zoom
│   │   ├── selection.ts
│   │   ├── lasso.ts
│   │   └── snap.ts
│   ├── items/
│   │   ├── base.ts
│   │   ├── text.ts                  ← existing
│   │   ├── image.ts                 ← existing
│   │   ├── frame.ts                 ← existing
│   │   ├── file.ts                  ← existing
│   │   ├── canvaslink.ts            ← existing
│   │   └── docwindow/
│   │       ├── index.ts             ← titlebar, resize, LOD swap
│   │       ├── md.ts                ← CodeMirror 6 + markdown
│   │       ├── code.ts              ← CodeMirror 6 + lang packs
│   │       ├── pdf.ts               ← PDF.js
│   │       ├── xlsx.ts              ← SheetJS / Univer
│   │       └── docx.ts              ← Mammoth (read-only)
│   ├── workspace/
│   │   ├── fs.ts                    ← FS adapter (FSAA → Tauri fs)
│   │   ├── manifest.ts              ← load/save workspace + canvas JSON
│   │   ├── viewstate.ts             ← per-file viewstate
│   │   └── index.ts                 ← search index management
│   ├── ui/
│   │   ├── sidebar.ts               ← outline + pinned tray
│   │   ├── topbar.ts
│   │   ├── search.ts
│   │   └── context-toolbar.ts
│   └── styles/
│       └── main.css
├── src-tauri/                       ← added during Tauri step
│   ├── Cargo.toml
│   ├── tauri.conf.json
│   ├── capabilities/main.json       ← fs scopes, dialog scopes
│   └── src/main.rs
├── package.json
├── vite.config.ts
└── README.md
```

## 6. Schemas (canonical)

### 6.1 `workspace.json`

```json
{
  "version": 2,
  "id": "ws_8f3e",
  "name": "Research",
  "createdAt": "2026-05-13T10:30:00Z",
  "updatedAt": "2026-05-13T14:22:00Z",
  "settings": {
    "theme": "light",
    "defaultCanvasId": "c_main",
    "lodThreshold": 0.3,
    "innerZoomDefault": 1.0,
    "autosaveDebounceMs": 500,
    "viewstateDebounceMs": 400
  },
  "pinned": [
    { "kind": "file",   "path": "notes/ideas.md" },
    { "kind": "folder", "path": "docs" }
  ]
}
```

### 6.2 `canvases/<id>.canvas.json`

```json
{
  "version": 2,
  "id": "c_main",
  "name": "Main",
  "viewport": { "x": 0, "y": 0, "zoom": 1.0 },
  "items": [
    {
      "id": "i_abc", "type": "text",
      "x": 120, "y": 80, "w": 280, "h": 140,
      "content": "<p>Some rich text...</p>",
      "color": "default", "tags": ["idea"],
      "pinned": false, "collapsed": false,
      "createdAt": "2026-05-12T...", "updatedAt": "..."
    },
    {
      "id": "i_img1", "type": "image",
      "x": 500, "y": 100, "w": 400, "h": 300,
      "assetRef": ".akash/assets/9a8b.png"
    },
    {
      "id": "i_frame1", "type": "frame",
      "x": 0, "y": 0, "w": 1200, "h": 800,
      "color": "warm", "locked": false
    },
    {
      "id": "i_file1", "type": "file",
      "x": 200, "y": 600, "w": 240, "h": 80,
      "path": "docs/spec.pdf", "label": "spec.pdf"
    },
    {
      "id": "i_link1", "type": "canvaslink",
      "x": 600, "y": 500, "w": 240, "h": 120,
      "target": {
        "workspaceId": "ws_abc",
        "canvasId": "c_main",
        "itemId": "i_xyz"
      }
    },
    {
      "id": "i_doc1", "type": "docwindow",
      "x": 800, "y": 300, "w": 600, "h": 400,
      "path": "notes/ideas.md",
      "kind": "md",
      "title": "ideas.md",
      "viewMode": "source",
      "innerZoom": 1.0,
      "showTitlebar": true,
      "tags": ["draft"]
    }
  ]
}
```

The `docwindow` item is the new type. It stores window-level state only — no file content, no cursor, no search query.

### 6.3 `viewstate/<pathhash>.json`

Path hash = `sha256(normalizedRelativePath).slice(0, 12)` where the path is normalized to forward slashes before hashing.

```json
{
  "path": "notes/ideas.md",
  "kind": "md",
  "updatedAt": "2026-05-13T14:22:00Z",
  "cursor": { "line": 42, "ch": 7 },
  "scrollTop": 1024,
  "selection": { "from": 100, "to": 150 },
  "search": { "query": "TODO", "caseSensitive": false },
  "viewMode": "source",
  "innerZoom": 1.0,
  "pdf":  { "page": 3, "rotation": 0 },
  "xlsx": { "sheet": "Sheet1", "scrollRow": 0, "scrollCol": 0 }
}
```

`pdf` and `xlsx` blocks present only when relevant.

## 7. In-memory data model

```typescript
type ItemId = string;
type CanvasId = string;
type DocKind = 'md' | 'code' | 'pdf' | 'xlsx' | 'docx';

type Item =
  | TextItem | ImageItem | FrameItem
  | FileItem | LinkItem | CanvasLinkItem
  | DocWindowItem;

interface DocWindowItem {
  id: ItemId;
  type: 'docwindow';
  x: number; y: number; w: number; h: number;
  path: string;
  kind: DocKind;
  title?: string;
  viewMode: 'source' | 'preview' | 'split';
  innerZoom: number;
  showTitlebar?: boolean;
  tags?: string[];
  pinned?: boolean;
}

interface WorkspaceState {
  workspace: Workspace;
  canvas: Canvas;
  items: Map<ItemId, Item>;
  selection: Set<ItemId>;
  // Runtime-only — not persisted:
  openDocs: Map<ItemId, OpenDocRuntime>;
  editorHandles: Map<ItemId, EditorHandle>;
  dirtyItems: Set<ItemId>;
  dirtyFiles: Set<string>;          // file paths needing fs flush
}

interface OpenDocRuntime {
  content?: string | Uint8Array;
  contentHash?: string;             // detect external edits
  mtimeOnLoad?: number;
  lodMode: 'live' | 'preview' | 'placeholder';
  isDirty: boolean;
}

type EditorHandle =
  | { kind: 'md'   | 'code'; view: EditorView /* CM6 */ }
  | { kind: 'pdf';  doc: PDFDocumentProxy }
  | { kind: 'xlsx'; book: WorkBook }
  | { kind: 'docx'; html: HTMLElement };
```

## 8. App-side storage (cross-workspace)

Lives in the app's IndexedDB on the web, or in a small SQLite file under the OS user-data directory under Tauri. Not inside any workspace folder.

```typescript
interface AppDB {
  recents: { path: string; name: string; lastOpenedAt: string }[];
  navStack: NavEntry[];             // existing back/forward state
  searchIndex: {
    [workspaceId: string]: {
      items: SearchEntry[];         // existing per-item index
      docs:  SearchEntry[];         // new: extracted from open-able files
      updatedAt: string;
    };
  };
  prefs: UserPrefs;
}

interface SearchEntry {
  itemId?: ItemId;                  // present for canvas items
  path?: string;                    // present for document content
  canvasId: CanvasId;
  kind: 'text' | 'doc' | 'note';
  text: string;
  range?: { line?: number; page?: number; sheet?: string; cell?: string };
}
```

**Content extraction per kind:**

- md / code → raw text
- pdf → `PDFDocumentProxy.getTextContent()` per page, joined with page boundaries
- xlsx → SheetJS flatten cells; record sheet + A1-style cell ref in `range`
- docx → Mammoth → plain text

Extraction triggers: first open of a file, on save (debounced), and on `Cmd/Ctrl+S` (immediate).

## 9. FS adapter contract

The single boundary between the app and the filesystem. One implementation today, swapped wholesale on the Tauri move. **Nothing above this layer changes during the Tauri migration.**

```typescript
interface Fs {
  readText(path: string): Promise<string>;
  writeText(path: string, content: string): Promise<void>;
  readBytes(path: string): Promise<Uint8Array>;
  writeBytes(path: string, data: Uint8Array): Promise<void>;
  list(dir: string): Promise<{ name: string; isDir: boolean }[]>;
  exists(path: string): Promise<boolean>;
  mtime(path: string): Promise<number>;
  pickWorkspace(): Promise<string | null>;
  watch(path: string, cb: (ev: FsEvent) => void): () => void;
}
type FsEvent = { kind: 'modified' | 'created' | 'removed'; path: string };
```

Implementation phases:

1. **Phase A — browser only.** Back this with the File System Access API. `watch` is a no-op (or polling-based). `pickWorkspace` uses `showDirectoryPicker`.
2. **Phase B — Tauri.** Re-back with `@tauri-apps/plugin-fs` + `plugin-dialog`. `watch` becomes real via `tauri-plugin-fs-watch` (or the watcher built into recent Tauri fs versions). Capability config restricts access to the user-chosen workspace directory.

## 10. Renderer contracts (one per file kind)

Each renderer is a module under `src/items/docwindow/` exposing:

```typescript
interface DocRenderer<TViewState> {
  kind: DocKind;
  mount(opts: MountOpts): Promise<RendererInstance<TViewState>>;
}

interface MountOpts {
  container: HTMLElement;           // the docwindow inner panel
  content: string | Uint8Array;     // file bytes already read by FS adapter
  viewstate: Partial<TViewState>;   // restored if available
  innerZoom: number;
  onDirty(): void;                  // editor signals content changed
  onViewstateChange(vs: Partial<TViewState>): void;
}

interface RendererInstance<TViewState> {
  setInnerZoom(z: number): void;
  setSearch(query: string, opts?: { caseSensitive?: boolean }): SearchResult[];
  jumpTo(ref: SearchResult | TViewState): void;
  getContent(): string | Uint8Array;  // for save
  getViewstate(): TViewState;
  destroy(): void;
}
```

Renderer-specific notes:

- **md / code (CM6):** `EditorView` with language pack chosen by file extension. Search via `@codemirror/search`. Inner zoom = CSS `transform: scale()` on the editor's outer element with `transform-origin: 0 0`; the editor itself keeps font metrics correct because glyph rendering is vector.
- **pdf (PDF.js):** virtualize pages, render each visible page to canvas at `dpr × innerZoom`. Debounce re-render on innerZoom change. Text-layer overlay enables find-in-document and selection.
- **xlsx (SheetJS for read, Univer for edit):** start read-only with SheetJS rendering into a virtualized grid; upgrade to Univer when editing semantics are needed.
- **docx (Mammoth):** convert to HTML once on mount, render inside a scrollable scaled container. No editing in v2.

## 11. Save lifecycle

```
keystroke / structural change
   └─► editor.onDirty()
        └─► OpenDocRuntime.isDirty = true
             ├─► UI: titlebar dot turns amber
             └─► debounce timer T (500ms by default)
                  └─► on timer fire:
                       1. content = renderer.getContent()
                       2. fs.writeText(path, content)
                       3. update mtime + contentHash on OpenDocRuntime
                       4. write viewstate (debounced separately, ~400ms)
                       5. update search index entry (background)
                       6. UI: dot → grey
```

**Cmd/Ctrl+S forces an immediate flush and synchronous index refresh** so the file shows up in global search instantly.

If any step fails: surface error in titlebar, keep `isDirty: true`, do not advance.

External edits (detected via mtime or fs watcher):

```
fs event → if (file's mtimeOnDisk > mtimeOnLoad):
              if (!isDirty):
                  silently reload content
              else:
                  prompt: "File changed on disk. Keep mine / Reload / Diff"
```

## 12. Search

Two layers:

- **In-document search** — driven by the renderer's `setSearch()`. Owns highlight, navigation, and match counts. Surfaced in the docwindow titlebar.
- **Global search** — your existing cross-workspace search, extended to include `docs` entries. Selecting a hit:
  - If the document is already a docwindow on the current canvas → focus it, zoom-to-fit, jump to match.
  - Else → spawn a new docwindow near the canvas center, jump to match.

Index refresh policy: on first open, on save, and on `Cmd/Ctrl+S` (immediate). Never block UI; run extraction in an idle callback or worker.

## 13. Zoom and LOD

**Two zoom levels:**

1. **Canvas zoom** — pan/zoom across the whole world. Applies a single `transform: scale(z) translate(x, y)` to `#world`.
2. **Inner zoom** — per-docwindow. Stored on the `DocWindowItem`. Applied as a `transform: scale(innerZoom)` on the inner editor panel. Controlled by:
   - Hover + `Cmd/Ctrl + scroll` inside the window
   - Zoom buttons in the docwindow titlebar
   - Keyboard `Cmd/Ctrl + =` / `Cmd/Ctrl + -` when focus is in the editor

**LOD swap thresholds** (configurable in `settings.lodThreshold`):

| Canvas zoom | docwindow mode | What renders |
|---|---|---|
| ≥ 0.6 | `live` | Full editor (CM6 / PDF.js / etc.) |
| 0.3–0.6 | `preview` | Static rendered content (no live editor) |
| < 0.3 | `placeholder` | Titlebar + filename + first-line snippet only |

Transitions destroy and recreate editor handles to keep memory bounded with many open windows.

## 14. Migration from v1 → v2

One-shot, forward-only migrator on workspace load:

1. If `.akash/workspace.json` exists with `version: 2`, load and proceed.
2. Else, if the old single manifest is present:
   - Back it up to `.akash/backups/pre-v2.json`.
   - Extract workspace-level fields (name, settings, recents-equivalent) into `.akash/workspace.json` with `version: 2`.
   - Extract items into `.akash/canvases/c_main.canvas.json` with the new shape; preserve item ids verbatim so cross-canvas links keep working.
   - Mark migration done.
3. Else, create a fresh empty workspace.

Migration must be idempotent: running it twice produces the same result. Never delete the backup automatically.

## 15. Open questions (deferred)

- **DOCX editing**: Mammoth is read-only. TipTap or OnlyOffice DocumentEditor are the realistic editing options if/when needed.
- **Linux WebKitGTK quirks**: known to lag Chrome/WKWebView on some rendering edge cases (CSS filters, certain font fallback). Plan a manual test pass on a Linux build before shipping.
- **External edit conflict UX**: above we sketch "Keep mine / Reload / Diff" — diff UI is non-trivial. v2 may ship without diff and just offer the binary choice.
- **Image and asset deduplication**: pasting the same image twice currently produces two assets. Content-hash naming would dedupe.
- **Large file thresholds**: at what size do we refuse to open a PDF or XLSX in-canvas and fall back to the existing external-open behavior? Likely 100 MB for PDF, 50 MB for XLSX.

## 16. Implementation milestones

| # | Milestone | Acceptance |
|---|---|---|
| M1 | FS adapter + manifest migration | Existing v1 workspaces load as v2 transparently. All existing item types work unchanged. |
| M2 | Pinned tray + docwindow chrome (no live editor) | Drag a `.md` file from the tray; a docwindow appears with titlebar showing filename + close. Resize, move, delete work. |
| M3 | Markdown docwindow (CM6) | The window contains a working CM6 editor in markdown mode. Edits autosave to file. Cmd/Ctrl+S forces flush. Cursor + scroll persisted in viewstate. |
| M4 | In-document search | Titlebar search field highlights matches, navigates next/prev. |
| M5 | Inner zoom + LOD | Inner-zoom controls work. Canvas-zoom thresholds swap to preview / placeholder. |
| M6 | PDF.js docwindow | Same shell, with PDF rendering and text-layer search. Page state in viewstate. |
| M7 | XLSX docwindow (read-only) | SheetJS render in virtualized grid. Sheet + scroll in viewstate. |
| M8 | Global search extended to doc contents | Hits from inside .md / .pdf / .xlsx surface in your existing search panel and can spawn or focus a docwindow. |
| M9 | Tauri shell | `tauri build` produces signed `.dmg` / `.msi` / `.AppImage` on CI. FS adapter swapped to Tauri plugin. File watcher reloads external edits. |
| M10 | Code docwindow (generic) | CM6 with language packs by extension. |
| M11 | DOCX docwindow (read-only) | Mammoth → HTML render. |

M1–M5 are the critical path to "useful." M6–M11 are progressive enhancement on the same docwindow shell.

---

*Spec frozen for implementation. Changes to schema or core contracts require a version bump on `workspace.json` and a migrator update.*
