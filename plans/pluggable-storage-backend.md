# Plan: Replace Snap!'s cloud-based saving with a pluggable storage backend

## Goal (as I understand it)

Make a variant of Snap! that saves/loads projects to **some other backend** instead
of the Berkeley Snap!Cloud, and simplify or remove the UI chrome that exists for the
current save/load/share flow.

This document has two halves:

1. **Open questions** — things I need from you before committing to a design. Please
   answer these inline (or tell me to make a call and proceed). The rest of the plan
   makes assumptions; the questions are where those assumptions could be wrong.
2. **The actual plan** — what the code looks like today, three implementation options
   ranked by effort, and a recommended path with concrete steps.

---

## Part 1 — Questions for you (please answer before we lock in)

### A. What is the new backend?
1. **What kind of store is it?** A REST API you control? A flat directory on a server?
   `localStorage`/IndexedDB only (purely local, no server)? The filesystem via the
   File System Access API? Google Drive / S3 / Dropbox? GitHub? Something embedding
   Snap! in a larger app that wants to own persistence (e.g. via `postMessage`)?
2. **Single-user or multi-user?** The current cloud has accounts, login, per-user
   project namespaces, sharing, and publishing. Does your backend need *any* of
   auth / multi-user / sharing, or is it "one user, just save my projects somewhere"?
3. **Protocol shape.** If it's an HTTP API, can you make it match a simple
   `save(name, blob)` / `load(name)` / `list()` / `delete(name)` contract, or are we
   adapting to a fixed external API we don't control?

### B. What should the UI look like?
4. **Keep a project browser, or go minimal?** Two ends of a spectrum:
   - **Minimal:** just "New / Open / Save / Save As" wired to the new backend; rip out
     the cloud login button, share/publish, the multi-tab project dialog.
   - **Full replacement:** keep the rich `ProjectDialogMorph` (thumbnails, notes,
     rename, delete, search) but point its "source" at the new backend.
5. **Should Import/Export to local disk stay?** These (`Export project...`,
   `Import...`, drag-and-drop of `.xml`) are independent of the cloud and very useful
   as an escape hatch. I'd recommend keeping them. Confirm?
6. **Do you need login UI at all?** If the backend has its own auth (or none), we can
   delete the cloud login/signup/account dialogs entirely.

### C. How "forked" do you want this?
7. **Is this a hard fork or do you want to stay mergeable with upstream Snap!?**
   This strongly affects the approach:
   - If you want to **track upstream**, we should minimize edits to existing files and
     lean on the config flags + a small new pluggable module (Option B).
   - If you **don't care about upstream merges**, we can rip out `cloud.js` and rewrite
     `save()`/`open` paths directly (Option A) — less code, more divergence.
8. **Deployment:** is this the same `snap.html` people open directly, or are you
   embedding Snap! inside another web app (iframe / same-page) that could pass a
   `config` object and even supply the storage adapter from outside?

### D. Data format
9. **Same project XML format, or a new envelope?** Snap! serializes projects to XML
   (`<project>…</project>`) plus media XML; the cloud wraps them as
   `{xml, media, thumbnail, notes, remixID}`. Easiest is to keep the exact same XML
   and just change *where the bytes go*. Any reason to change the format itself? (e.g.
   you want JSON, or you want media inline.)

My **default assumptions** if you don't answer (so the plan can proceed): a single-user
HTTP backend we control with a simple `save/load/list/delete` contract, no sharing/
publishing, keep local Import/Export, stay reasonably mergeable with upstream, same XML
format.

---

## Part 2 — How saving/loading works today

(Findings from reading `src/cloud.js`, `src/gui.js`, `src/store.js`, `src/api.js`,
`docs/API.md`, `docs/Extensions.md`. Line numbers are approximate and current as of
this branch.)

### The Cloud class (`src/cloud.js`)
- `Cloud` is a ~1100-line class instantiated once: `this.cloud = new Cloud();` in
  `IDE_Morph.init` (`gui.js:306`).
- It talks to Snap!Cloud over `XMLHttpRequest` via a single low-level
  `Cloud.request(method, path, onSuccess, onError, …)` (`cloud.js:163`). All higher-level
  methods (`saveProject`, `getProject`, `getProjectList`, `deleteProject`,
  `shareProject`, `publishProject`, `login`, `logout`, `checkCredentials`, …) build on it.
- The backend URL is auto-detected (`determineCloudDomain`, `cloud.js:74`) from a
  `<meta name='snap-cloud-domain'>` tag or `location.host`, falling back to
  `https://snap.berkeley.edu`. API paths like `/api/v1/projects/%username/{name}` are
  hardcoded, with `%username` substituted in `request()`.
- **`Cloud.disable()`** (`cloud.js:56`) sets `this.disabled = true` and clears the
  username. This already exists and is the cleanest seam for turning the cloud off.

### How the IDE talks to the cloud (`src/gui.js`)
- **`save()`** (`gui.js:6125`) is the router. Logic:
  - On `file:` protocol → always export to disk.
  - Else, based on `this.source`: `'disk'` → `exportProject()`; `'cloud'` →
    `saveProjectToCloud()`; otherwise → open the `ProjectDialogMorph` ("Save As").
  - Note `if (this.cloud.disabled) {this.source = 'disk';}` (`gui.js:6148`) — disabling
    the cloud already makes Save fall back to disk export.
- **`saveProjectToCloud()`** (`gui.js:9568`) builds the request body
  (`{xml, media, thumbnail, notes, remixID}` via `buildProjectRequest`, `gui.js:9514`)
  and calls `cloud.saveProject(...)`.
- **Opening:** `ProjectDialogMorph` (`gui.js:~10100+`) is the multi-source open/save
  dialog. Its source can be `'cloud'`, `'examples'`, `'local'` (deprecated localStorage),
  or `'disk'`. For cloud it calls `cloud.getProjectList()` / `getProject()`.
- **`this.source`** (a string on the IDE) tracks where the current project came from.
  This is the existing abstraction for "which backend" — we can extend it.

### Serialization seams (`src/store.js`, `src/api.js`) — backend-agnostic
- Serialize: `ide.serializer.serialize(new Project(this.scenes, this.scene))` →
  XML string. Public API wrapper: `IDE_Morph.getProjectXML()` (`api.js:239`).
- Deserialize/load: `ide.openProjectString(str)` → `rawOpenProjectString` →
  `serializer.load(...)`. Public API wrapper: `IDE_Morph.loadProjectXML(xml)` (`api.js:243`).
- These are clean and **a new backend only needs these two calls** — it never touches XML internals.

### Local (non-cloud) save/load — reusable as-is
- Export to disk: `exportProject()` → `saveXMLAs()` → `saveFileAs()` (uses FileSaver.js,
  `gui.js:7334`).
- Import from disk: `importLocalFile()` (`gui.js:5403`, hidden `<input type=file>`) and
  drag-and-drop `droppedText()` (`gui.js:3295`), which sniffs `<project>` / `<snapdata>` /
  `<blocks>` / CSV / JSON.

### localStorage / settings — already present
- Settings persisted under `-snap-setting-*`; an unsaved-project **backup** under
  `-snap-backup-*` (`backup`/`restore`/`clearBackup`, `gui.js:3913+`); deprecated
  per-project storage under `-snap-project-*`. No IndexedDB usage today. A localStorage/
  IndexedDB backend could reuse these helpers.

### Existing config flags to hide UI chrome (the good news)
`IDE_Morph` takes a **`config` object** (`new IDE_Morph(config)`, `gui.js:255`), stored as
`this.config`, applied in `buildPanes` / `applyPaneHidingConfigurations`
(`gui.js:870–1037`). URL hash params also feed into the same flags
(`gui.js:471–499`). Relevant keys:

| Flag | Effect | Source |
|---|---|---|
| `noCloud` | Calls `cloud.disable()` — turns off all cloud access | `gui.js:471, 988` |
| `hideCloudMenu` | Hides the cloud button in the toolbar | `gui.js:498, 1601` |
| `hideProjects` | Hides the project (file) menu button | `gui.js:1597` |
| `noProjectItems` | Removes New/Open/Save/etc. from the project menu | `gui.js:5067` |
| `noShare` | Removes share/publish from the project dialog | `gui.js:496, 10779` |
| `hideControls` | Hides the whole toolbar | `gui.js:1003` |
| `hideProjectName` | Hides the project title in the toolbar | `gui.js:1655` |
| `noImports` | Disables drag-and-drop import | `gui.js:3133+` |
| `noExitWarning` | Skips the unsaved-changes close warning | `gui.js:994` |

**Takeaway:** A large chunk of "remove the save/load chrome" is already achievable with
flags — no surgery required. `noCloud` alone disables the cloud and makes Save export to
disk. The work is mainly (a) wiring Save/Open to the *new* backend and (b) deciding how
much of the dialog UI to keep.

---

## Part 3 — Implementation options

### Option A — Direct replacement (simplest, most divergent)
Gut `cloud.js`'s role: set `noCloud`, then override `save()` and the open path to call
your backend's `save(name, xml)` / `load(name)` directly. Add a thin "Open" menu that
lists `backend.list()`. Delete/hide all cloud UI via flags.

- **Pros:** least new abstraction; fastest to a working prototype; easy to read.
- **Cons:** edits core `gui.js` methods directly → painful upstream merges; not reusable
  if you later want multiple backends.
- **Good if:** hard fork, single backend, single user.

### Option B — Pluggable `StorageBackend` interface (recommended)
Introduce a small interface and make the IDE talk to *it* instead of to `Cloud` directly.

```
StorageBackend (new file, e.g. src/storage.js)
  .name                       // label for menus
  .capabilities               // {list, delete, rename, thumbnails, auth, share}
  .save(name, projectBlob)    // projectBlob = {xml, media, thumbnail, notes}
  .load(name) -> projectBlob
  .list() -> [{name, updated, thumbnail?}]
  .delete(name)
  .rename(old, new)           // optional
```

Then:
1. Provide `CloudBackend` (wraps today's `Cloud`) and your `XyzBackend`.
2. Let the active backend be chosen via a new `config.storage` (an adapter object or a
   name string). Default = `CloudBackend` so upstream behaviour is unchanged.
3. Refactor `save()` / `saveProjectToCloud()` and the open path to call
   `ide.storage.save/load/list` instead of `ide.cloud.*`. Keep `this.source` semantics
   but add a `'backend'` source.
4. The `ProjectDialogMorph` "cloud" source becomes "the active backend"; gate
   capability-specific UI (share/publish/auth) on `backend.capabilities`.

- **Pros:** clean seam; backends are swappable and testable; minimal change to
  serialization; stays much closer to upstream (most edits are localized + additive);
  you can ship multiple backends (localStorage, HTTP, Drive…).
- **Cons:** more upfront design; touching `ProjectDialogMorph` is the fiddly part.
- **Good if:** you want this to last, possibly support more than one backend, and stay
  mergeable.

### Option C — Stay 100% external via the embedding API (zero core edits)
Don't modify Snap! at all. Embed it (iframe or same page), hide chrome with config
flags (`noCloud`, `hideControls`, `noProjectItems`, …), and drive save/load from the
host page using the documented API: `getProjectXML()` / `loadProjectXML()` /
`unsavedChanges()` (`docs/API.md`). Your host app owns all persistence UI.

- **Pros:** literally no fork; trivially tracks upstream; clean separation.
- **Cons:** you must build your own UI in the host; in-IDE "Save" button won't natively
  hit your backend unless you also override it; only works if you control an embedding page.
- **Good if:** Snap! is embedded inside a larger app you're building.

---

## Recommended path

**Option B** if this is a standalone Snap! you'll maintain, **Option C** if you're
embedding Snap! in your own app. Both start with the same cheap first step, so we can
defer the big decision:

### Phase 0 — Prove the seam (½ day, reversible)
- Launch with `config = {noCloud: true, hideCloudMenu: true, noShare: true}` and confirm
  the cloud disappears and Save falls back to disk export. (Edit the `new IDE_Morph()`
  call in `snap.html:~59`, or pass via URL hash.) This validates how much the flags buy
  us before writing any backend code.

### Phase 1 — Backend interface + one real backend (Option B)
- Add `src/storage.js` with the `StorageBackend` interface + a `CloudBackend` wrapper
  (so default behaviour is unchanged) + your new backend.
- Add `config.storage`; instantiate `this.storage` in `IDE_Morph.init` next to `this.cloud`.
- Refactor `save()`, `saveProjectToCloud()` (rename to `saveProjectToBackend`), and the
  open/list path to go through `this.storage`. Keep disk Import/Export untouched.
- Load `storage.js` in `snap.html` (respect the documented load-order + `?version=` cache-bust).

### Phase 2 — Trim the UI
- Decide minimal vs. full dialog (Question 4). For minimal: rely on `noShare`,
  `hideCloudMenu`, and a simplified Open menu listing `storage.list()`. For full: adapt
  `ProjectDialogMorph` to render from the active backend and gate share/auth on
  `capabilities`.
- Remove now-dead login/signup/account dialogs if the backend needs no auth (Question 6).

### Phase 3 — Polish
- Unsaved-changes / backup behaviour (`recordSavedChanges`, the `-snap-backup-*` flow)
  still works since it's backend-agnostic — verify it.
- Update `HISTORY.md`; bump version triple if releasing (`SnapVersion` in `gui.js`,
  `snapVersion` in `sw.js`, `?version=` strings in `snap.html`) per `CLAUDE.md`.

### Risks / watch-outs
- `ProjectDialogMorph` is the most entangled piece (multi-source tabs, thumbnails,
  remix/share). Touching it is where most of the effort and risk lives — its scope
  depends heavily on Question 4.
- The `file:` protocol special-case in `save()` overrides everything; if you test by
  opening `snap.html` directly you'll always get disk export. Serve over HTTP to
  exercise the backend (`python3 -m http.server`).
- `this.source` is referenced in several places (`gui.js:5943, 6148, 7925, 9325, 10128,
  10174, 10178, 10545`) — keep its state machine coherent when adding a backend source.

---

## TL;DR
There are real hooks already: a `config` object that can hide essentially all the
save/load/cloud chrome (`noCloud`, `hideCloudMenu`, `noShare`, `noProjectItems`,
`hideControls`…), a clean `Cloud.disable()`, and backend-neutral serialization
(`getProjectXML` / `loadProjectXML`). What's *missing* is an abstraction so the IDE can
save to something other than `Cloud` — that's the `StorageBackend` interface proposed in
Option B. **But before I build it, please answer the Part 1 questions — especially what
the backend actually is (A1–A3) and how much project-browser UI you want to keep (B4).**
