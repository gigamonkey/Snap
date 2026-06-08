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

### A Answers
1. I'm planning to embed Snap! in my class website where I already have
   infrastructure for saving code students write in the browser to Github. So
   for our purposes assume I have existing Javascript code that runs in the
   browser that can store stuff the way I want.
2. In the class website students are OAuth logged in to Google and Github and I save
   their work in a repo that I set up for them.
3. I control the server so we can adapt it however we need.

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

### B Answers
4. I think possibly both. In some context I will just want them to write some
   Snap! in a contex I provide and then I will take care of saving it. (With my
   Java and Javascript code I save the code to git every time they run it. But I
   will also probably want to have something more like the current Snap! project
   browser for more complex assignments.
5. Sure. They will also provide a good sanity test: can we import a project that
   was exported from regular Snap! and can regular Snap! import a project we
   export.
6. You can assume I have already handled logging students into my website.

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

### C answers
7. I definitely want to track upstream and even better, I'd like to contribute
   changes upstream so I'm using officially supported APIs.
8. I'm going to embed it in another web app. Using in iframe is fine as I
   already do that for Javascript. Or same page if that's easier.

### D. Data format
9. **Same project XML format, or a new envelope?** Snap! serializes projects to XML
   (`<project>…</project>`) plus media XML; the cloud wraps them as
   `{xml, media, thumbnail, notes, remixID}`. Easiest is to keep the exact same XML
   and just change *where the bytes go*. Any reason to change the format itself? (e.g.
   you want JSON, or you want media inline.)

## D answers
9. For now assume we're keeping the exact same XML.

---

## Part 1.5 — Decision summary (synthesized from your answers)

Your answers point at a meaningfully different shape than the original defaults, so here
is the direction the rest of the plan now assumes:

- **The backend is host-supplied JS, not a Snap!-internal HTTP client.** You already have
  browser code that persists student work to per-student GitHub repos, and the website
  handles OAuth/login. So Snap! should *not* learn about GitHub at all — it should expose
  a seam that lets your existing JS act as the storage backend. (This is also exactly the
  kind of generalization that's plausible to land upstream.)
- **No auth / login / sharing UI inside Snap!.** Your site owns identity. We delete or
  hide the cloud login/signup/account/share UI.
- **Two usage modes, both supported by the same seam:**
  1. **Minimal / host-owned:** embed Snap! with the chrome hidden; your page reads the
     project with the public API (`getProjectXML()`) on your own cadence (e.g. on run)
     and writes it back with `loadProjectXML()`. Snap!'s own Save does nothing visible.
  2. **In-IDE browser:** for richer assignments, keep a Snap!-style project dialog
     (open / save / list / delete) but point it at your injected backend instead of the
     cloud.
- **Keep local Import/Export** (also serves as the round-trip compatibility test: a
  project exported by stock Snap! must import here, and vice-versa).
- **Same project XML format**, unchanged.
- **Track upstream, and aim to contribute the seam upstream** using officially supported
  APIs. This makes "minimize edits to existing files; add a clean, documented extension
  point" a hard requirement, and adds an explicit *align-with-maintainers* step before
  building.
- **Embedding:** iframe or same-page. Because you need to hand Snap! a *live JS adapter
  object*, this works cleanly only **same-origin** (serve Snap! from your own domain, or
  same page). A cross-origin iframe can't receive a JS object and would force a
  `postMessage` bridge — more code and harder to standardize. **Recommendation:
  same-origin iframe (or same page).**

This makes the recommended approach a **hybrid of Option B (a host-injectable
`StorageBackend`) and Option C (the embedding API)** — see the revised Recommended path
below. The default assumptions from the earlier draft are superseded by the above.

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

## Recommended path (revised for your answers)

Chosen approach: **a host-injectable `StorageBackend` (Option B) whose adapter is supplied
by your embedding page, combined with the existing embedding API (Option C) for the
minimal mode.** Built to be contributed upstream, so every phase prefers *adding a clean,
documented extension point* over editing core flow.

The phases are ordered so that the two early ones deliver your **minimal mode with zero
core changes** — usable immediately and trivially upstream-safe — and the later ones add
the in-IDE browser, which is where the real (contributable) refactor lives.

### Phase 0 — Embed + hide chrome, host owns saving (minimal mode, no core edits)
Goal: a working embedded Snap! where your page does all the persistence.
- Serve Snap! **same-origin** with your site (so you can reach the live IDE object).
- Launch with `config` flags to strip the save/load/cloud chrome:
  `{noCloud: true, hideCloudMenu: true, noShare: true, hideProjects: true,
  noProjectItems: true, hideProjectName: true, noExitWarning: true}`
  (tune to taste — e.g. keep `hideProjects` off if you still want Import/Export in a menu).
- From your page, get the IDE (`iframe.contentWindow.world.children[0]`, per `docs/API.md`)
  and drive persistence with the **public API**: read with `getProjectXML()`, write with
  `loadProjectXML(xml)`, and use `unsavedChanges()` / `resetUnsavedChanges()` to decide when.
- Wire your "save on run" behaviour from the host side. **Open item:** confirm whether you
  want to save on Snap!'s green-flag run specifically — if so, we may need a small official
  hook (an `onrun`/save callback) since the public API doesn't currently expose a run event.
  Saving on a timer or on your own button needs no core change.

Deliverable: students can write & run Snap! embedded in your site; you persist to GitHub
exactly as you do for JS today. **No fork of Snap! source at all.**

### Phase 1 — Define the `StorageBackend` contract (design + upstream alignment)
Before writing the in-IDE browser, pin the interface and get maintainer buy-in (your
explicit goal of contributing upstream makes this the highest-leverage step).
- Draft the adapter contract (host implements it; defaults to today's `Cloud`):

```
StorageBackend
  .name                       // label shown in menus
  .capabilities               // {list, delete, rename, thumbnails}  (no auth/share for you)
  .save(name, projectBlob)    // projectBlob = {xml, media, thumbnail, notes}; -> Promise
  .load(name) -> Promise<projectBlob>
  .list() -> Promise<[{name, updated, thumbnail?, notes?}]>
  .delete(name) -> Promise
  .rename(old, new) -> Promise   // optional; present only if capabilities.rename
```

- Decide the injection mechanism that maintainers would accept — most likely a new
  `config.storage` (an object the host passes), with `Cloud` refactored to *implement*
  this same contract so it's a true generalization, not a parallel path.
- **Action:** raise this on the Snap! forum / with Jens & Brian (a short design note +
  this contract) before building, so the implementation matches what they'll merge.

### Phase 2 — Make the IDE talk to `this.storage` instead of `this.cloud`
- Add `src/storage.js`; instantiate `this.storage` in `IDE_Morph.init` next to `this.cloud`
  (defaulting to a `Cloud`-backed adapter so stock behaviour is byte-for-byte unchanged).
- Route `save()` / `saveProjectToCloud()` and the open/list/delete paths through
  `this.storage`. Keep `this.source`'s state machine coherent (add a `'backend'` source);
  leave disk Import/Export untouched.
- Load `storage.js` in `snap.html` in dependency order, with a `?version=` cache-bust string.

### Phase 3 — In-IDE project browser pointed at the injected backend
- Adapt `ProjectDialogMorph` so its "cloud" source becomes "the active backend," rendering
  the list from `storage.list()` and gating capability-specific UI on `capabilities`
  (so share/publish/auth simply don't appear for your backend).
- Remove the now-dead login/signup/account dialogs from this build (you confirmed no
  in-Snap! login is needed).

### Phase 4 — Compatibility test + polish
- **Round-trip test (your suggested sanity check):** export a project from stock Snap!,
  import it here; export from here, import into stock Snap!. Must be lossless — this is the
  acceptance test for "same XML format."
- Verify unsaved-changes/backup (`recordSavedChanges`, `-snap-backup-*`) still behaves —
  it's backend-agnostic but worth confirming with the new save path.
- Update `HISTORY.md`; if you cut a build, bump the version triple (`SnapVersion` in
  `gui.js`, `snapVersion` in `sw.js`, `?version=` strings in `snap.html`) per `CLAUDE.md`.

### Risks / watch-outs
- **Cross-origin embedding breaks object injection.** Handing Snap! a live adapter object
  (and reaching `contentWindow.world…`) requires same-origin. If Snap! must be served
  cross-origin, we'd need a `postMessage` storage bridge — more work; flag early.
- `ProjectDialogMorph` is the most entangled piece (multi-source tabs, thumbnails,
  remix/share). Phase 3 is where most effort/risk lives; Phases 0–2 deliver value without it.
- The `file:` protocol special-case in `save()` (`gui.js:6125`) overrides everything —
  test over HTTP (`python3 -m http.server`), not by opening `snap.html` directly.
- `this.source` is referenced across `gui.js` (`5943, 6148, 7925, 9325, 10128, 10174,
  10178, 10545`) — keep it consistent when adding the backend source.
- **Upstream acceptance is not guaranteed.** Phase 1 alignment de-risks this, but plan for
  the possibility of carrying the `storage.js` seam as a thin local patch if a PR stalls.

### Open items still worth confirming
- **Save-on-run hook:** do you specifically need to persist on the green-flag run? If yes,
  we likely add a small official callback (Phase 0 open item).
- **Same-origin hosting:** can you serve Snap! from your own domain (or same page)? If not,
  we switch to the `postMessage` bridge variant.
- **List metadata:** what does your GitHub-backed `list()` return cheaply (names only? dates?
  thumbnails)? This sets how rich the Phase 3 browser can be without extra round-trips.

---

## TL;DR
Your case is **embed Snap! in your site, hide its save/load chrome, and let your existing
GitHub-saving JS be the backend — using official, upstream-friendly APIs.** Good news:
**Phases 0–1 give you the minimal mode with no fork at all** (config flags +
`getProjectXML`/`loadProjectXML`), so you can be running embedded immediately. The fuller
in-IDE project browser needs a real but contained change — a host-injectable
`StorageBackend` that generalizes today's `Cloud` — which we should socialize with the
maintainers (Phase 1) before building so it can land upstream. Keep everything same-origin,
keep the XML format, keep Import/Export as the round-trip compatibility test.
