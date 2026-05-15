# Ak@sh — Development Guide

Source of truth for architecture and schemas: **SPEC.md** (v2 design spec).

## Common commands

```bash
# Dev server (placeholder — Vite not yet wired)
npm run dev

# Build (placeholder — Tauri not yet wired)
npm run build

# Test (placeholder — no test runner yet)
npm test
```

## Conventions

- No large refactors without explicit ask.
- Preserve existing item `id` values across migrations — cross-canvas links depend on them.
- The FS adapter (SPEC section 9) is the **only** place anything touches the filesystem. All other code goes through its interface.
- Any change to SPEC.md schemas or core contracts requires a `version` bump in `workspace.json` and a corresponding migrator update.

## Boundary-conversion pattern (M1)

On disk the app uses **v2 format** (`.akash/workspace.json` + `.akash/canvases/*.canvas.json`). In memory it uses **v1 format** (`width`/`height`, `modifiedAt` epoch, `bgColor`, flat canvaslink fields). Conversion happens at the `loadCanvas`/`saveCanvas` boundary in `FolderStorage` via `itemV1toV2` / `itemV2toV1` and `viewportV1toV2` / `viewportV2toV1`. All rendering, event, undo, and search code sees v1 shapes — zero UI code changes.

### Deferred SPEC fields (not yet converted)

| SPEC v2 field | v1 field | Reason |
|---|---|---|
| Image `assetRef` | `file` (in `images/` dir) | Requires blob move to `.akash/assets/` |
| File `path` + `label` | `file` + `originalName` | SPEC references in-place; v1 copies into `files/` |
| Text `content` (HTML only) | `content` + `contentHtml` | Rendering uses both; merge touches UI code |
| Canvaslink display fields dropped | `targetWorkspaceName`, `targetItemPreview`, `targetItemType` | Rendering reads these; kept alongside nested `target` |
