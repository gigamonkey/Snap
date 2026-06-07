# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

Snap! (formerly BYOB) — a visual, blocks-based programming language and IDE that runs entirely in the browser. Written by Jens Mönig and Brian Harvey (UC Berkeley). This repo is the client-side IDE only; the project-sharing backend lives in a separate repo (snap-cloud/snapCloud).

## No build system

There is **no build step, no bundler, no package.json, no test suite, and no dependencies to install.** The app is plain ES6 loaded directly via `<script>` tags.

- **Run / test locally:** open `snap.html` in a browser. `index.html` just redirects to it. To exercise file/cloud features you usually need to serve over HTTP rather than `file://` — e.g. `python3 -m http.server` then open `http://localhost:8000/snap.html`.
- **"Testing" means** playing with your change in the running IDE. There are no unit tests. Right-click any morph in the running app to open a Smalltalk-style inspector with a JS evaluation pane (`this` = the inspected object) — the primary debugging tool.
- **Linting:** code is expected to pass JSLint (jslint.com) with: assume browser, tolerate missing `'use strict'`, indentation 4, max line length 78. Core morphic.js additionally tolerates eval and unfiltered for-in.

## Source loading order (matters)

`src/*.js` files are loaded in a fixed order in `snap.html` because they depend on globals defined by earlier files. The dependency chain bottoms out at morphic.js:

```
morphic.js  → base GUI framework (the "Morphic" world; all UI is a Morph)
  widgets.js, symbols.js  → dialogs, buttons, vector-icon glyphs
  blocks.js   → visual block rendering & interaction (SyntaxElementMorph, BlockMorph, ...)
  threads.js  → the evaluator: Process / ThreadManager that run block scripts
  objects.js  → SpriteMorph, StageMorph, and the primitive block definitions
  scenes.js   → multi-scene support
  gui.js      → IDE_Morph — the top-level application shell, menus, palettes, corral
  byob.js     → "Build Your Own Blocks": custom block definition & editor
  lists.js, tables.js  → Snap! list data type and table view
  store.js    → XML serialization (load/save projects); needs xml.js
  ...plus paint.js, sketch.js, video.js, maps.js, extensions.js, cloud.js, api.js, embroider.js
```

Each module stamps a `modules.<name> = '<date>'` global at the bottom and declares its prerequisites in the header comment block. Read that header block before editing a file — it lists the constructor hierarchy and what the module needs.

## Architecture notes

- **Everything is a Morph.** The UI is the Morphic framework (morphic.js): a tree of `Morph` objects rendered to a single `<canvas>`. There is no DOM-based UI and no virtual DOM. Construct UI by composing Morphs, not HTML. `morphic.txt` (in `docs/`) is the full Morphic manual.
- **Three layers of "code":**
  1. JavaScript source (`src/`) implements the IDE and the *primitive* blocks.
  2. Block scripts authored by users are evaluated by threads.js (`Process`), not compiled to JS.
  3. Increasingly, parts of Snap! itself are **bootstrapped** — written in Snap! blocks and represented in a Lisp-like text syntax. See `docs/Syntax.md`. Look for `bootstrappedBlocks` / `blockMigrations` in objects.js.
- **Class system:** hand-rolled prototype inheritance via Morphic's constructor pattern (`SomeMorph.uber.init.call(this)` for super-calls), **not** ES classes. Per the contributing guide, initialize all instance attributes in the constructor/`init()` so the object's shape is visible at a glance.
- **Serialization:** projects, sprites, and blocks save/load as XML through store.js (`SnapSerializer`) + xml.js. `ypr.js` handles legacy Scratch/BYOB import.
- **Media & libraries** are shipped as files, not code: `Costumes/`, `Backgrounds/`, `Sounds/` (with `*.json` index manifests), `libraries/` (XML block libraries + a `LIBRARIES` index), `Examples/`, `help/`. The `Costumes/COSTUMES.json`-style manifests list what the media browser shows.

## Conventions the maintainers enforce (from docs/CONTRIBUTING.md)

- **One global "World", unique names for everything.** Avoid modules/namespaces/IIFEs, frameworks (no jQuery), and direct DOM access.
- Use `var myself = this;` to capture `this` for inner scopes rather than passing `thisArg` to `map`/`filter`/`forEach` (except `call`).
- Avoid: nested ternaries, RegEx, giant functions (they're "modules in disguise" — make a new constructor instead), and overwriting existing definitions.
- Commits favor readability over cleverness — many small, interchangeable code chunks.

## Versioning

Two places must be kept in sync when cutting a release, plus the cache-busting query strings:

- `SnapVersion` in `src/gui.js`
- `snapVersion` in `sw.js` (also drives the PWA cache name and the list of files cached offline)
- `?version=YYYY-MM-DD` query strings on each `<script>` in `snap.html` (cache-busting per file)

## Internationalization

Translations live in `locale/lang-<code>.js`, each assigning to `SnapTranslator.dict.<code>`. Use `lang-de.js` as the template. Block specs use ` _ ` (underscore surrounded by spaces) as input placeholders; translated specs must preserve the order and count of underscores. Launch a translated build with `?lang=<code>` in the URL. See the translation section of `docs/CONTRIBUTING.md`.

## Docs worth reading

`docs/API.md` (embedding/automation via `IDE_Morph` methods), `docs/Extensions.md` (JS extension blocks), `docs/Migrating.md` (Morphic 2 / v6 migration), `docs/Offline.md` (PWA), `docs/Syntax.md` (Lisp notation for blocks), `docs/morphic.txt` (Morphic framework manual). `HISTORY.md` is the running changelog (top "in development" section first).
