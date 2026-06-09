# Agent guide — floor3dpro-card

Tool-agnostic guide for coding agents (Cursor, Claude Code, Copilot, Codex, …).
This is the only `AGENTS.md` in the repo — it's small enough that repo-wide
orientation lives here. Add a nested file only if a subfolder grows its own
distinct conventions.

## What this is

A Home Assistant **Lovelace card** that renders a 3D "digital twin" of a home:
a single `<canvas>` Three.js scene where 3D model objects are bound to HA
entities (lights illuminate, doors/covers animate open/closed, sensors show
text, materials recolor by state). The user supplies an OBJ (+MTL) or GLB/glTF
model exported from Sweet Home 3D / Blender / SketchUp / etc., and the card
config maps named mesh objects to entities.

- **Stack**: TypeScript + [Lit](https://lit.dev) 2 (`LitElement`, decorators) +
  [Three.js](https://threejs.org) `0.130` (OrbitControls, OBJ/MTL/GLTF loaders,
  Sky) + `@tweenjs/tween.js` for animation + `@material/mwc-*` web components for
  the editor UI. `custom-card-helpers` provides HA types/action handling.
- **Custom element tag** (the Lovelace `type:`): **`custom:floor3dpro-card`**
  (element `floor3dpro-card`, editor `floor3dpro-card-editor`). Defined in
  `src/floor3dpro-card.ts` via `@customElement('floor3dpro-card')` and
  registered in `window.customCards`.
- **Distribution**: HACS plugin (`hacs.json`, category `plugin`). The built
  bundle is `dist/floor3dpro-card.js` (committed). Consumers register it as a
  Lovelace dashboard resource.
- **Fork status**: this is a **fork of `andyHA`/Andrea Di Zanni's
  [`floor3d-card`](https://github.com/adizanni/floor3d-card)**, renamed to the
  `floor3dpro-card` element/tag and re-themed as the "Game Engine Backbone
  Edition". Local "Pro" additions over upstream (search the source for the
  marker comments `Faz-0`/`Faz-1` and `pro`): a deterministic render
  scheduler, a shared asset cache with per-instance deep clones
  (`__assetCache*` in `floor3dpro-card.ts`), a transactional (draft/commit)
  editor, DOM custom-element isolation (the `floor3dpro-*` tag prefix), and
  opt-in `pro_skill` (`level`/`editor`/`mobile`/`all`) + `pro_log`
  (`engine`/`all`) config (both opt-in, empty by default; see `src/helpers.ts`
  `proGetSkillSet`/`proGetLogSet`). Architecture
  notes live in `ENGINE.md`. The upstream remote is the repo's `repository`
  field (`levonisyas/floor3dpro-card`); this workspace clone's `origin` is
  `nantomarioni/floor3dpro-card`.

> Version is tracked in two places that must agree: `package.json` `version`
> and `CARD_VERSION` in `src/const.ts` (currently `1.5.3-Pro.Faz.2.1`).

## How to run things

Package manager is **npm** (`package-lock.json` is the live lockfile;
`yarn.lock.backup` is a stale artifact — ignore it). Node ≥ 16 era toolchain
(Rollup 2 / TypeScript 4). Install with `npm ci` (or `npm install`).

```bash
npm run start    # rollup -c rollup.config.dev.js --watch
                 #   → rebuilds to dist/ and serves on http://0.0.0.0:5000 (CORS open)
npm run build    # lint THEN rollup -c (the release build; minified via terser)
npm run rollup   # rollup -c only (build the bundle without linting)
npm run lint     # eslint src/*.ts
```

There is **no typecheck script, no formatter script, and no tests**. Notes:

- `tsconfig.json` sets `"noEmit": true`; type-checking happens implicitly via
  `rollup-plugin-typescript2` during the build, not as a standalone gate.
- Prettier is configured (`.prettierrc.js`: semi, single quotes, trailing
  commas, width 120, 2-space) and wired into ESLint, but there is no
  `npm run format`. Run `npx prettier` manually if you need to format.
- For local HA testing, point a Lovelace resource at the dev server URL
  (`http://<host>:5000/floor3dpro-card.js`) or drop the built
  `dist/floor3dpro-card.js` into `config/www/`.

## Quality gate

CI (`.github/workflows/hacs.yml`) only runs the **HACS validation action**
(`hacs/action@main`, category `plugin`) — it checks repo/HACS metadata, NOT
lint or build. So the practical local gate before declaring a change done:

1. `npm run lint` — clean.
2. `npm run build` — succeeds (this also runs lint + type-check via the Rollup
   TS plugin and regenerates `dist/floor3dpro-card.js`).
3. **Commit the rebuilt `dist/floor3dpro-card.js`** — it's the distributed
   artifact and is tracked in git. A source change without a matching `dist`
   rebuild ships stale code to HACS users.
4. Keep `package.json` `version` and `src/const.ts` `CARD_VERSION` in sync.

## Conventions

- **Single-element architecture.** Almost everything lives in one big
  `LitElement` (`src/floor3dpro-card.ts`, ~4k lines): it owns the Three.js
  `Scene`/`Camera`/`WebGLRenderer`/`OrbitControls`, the model load, the
  per-entity state→3D mapping, and the render loop. Don't reach for a framework
  refactor; match the existing imperative style.
- **Lit conventions**: `@customElement`, `@state`/`@property` decorators
  (`experimentalDecorators` on), `html`/`css` tagged templates. HA wires the
  card by calling the static `getConfigElement()` (lazy-imports `./editor`),
  static `getStubConfig()` (default config in the card picker), and the instance
  `setConfig(config)` / `getCardSize()`.
- **Tag prefix isolation**: every locally-defined element is prefixed
  `floor3dpro-` (`floor3dpro-card`, `floor3dpro-card-editor`, and the
  `elements/*.ts` MWC wrappers `floor3dpro-button|select|textfield|formfield`)
  so this fork can coexist with the upstream `floor3d-card` on the same page.
  Preserve this when adding elements.
- **Config typing**: `Floor3dCardConfig` in `src/types.ts` is the master config
  shape (intentionally loose — many `any`s, `strictNullChecks: false`,
  `noImplicitAny: false`). Per-entity binding is `EntityFloor3dCardConfig`
  (`entity`, `type3d`, `object_id`, …). Add new config options to `types.ts`
  and to `getStubConfig()` defaults where appropriate.

### The object-name ↔ entity mapping model (the core idea)

This is the contract the whole card is built around — understand it before
touching binding logic.

- A config has a single 3D model (`objfile` (+`mtlfile`) or a `.glb`, under
  `path`) and a list of `entities`. Each entry binds **one HA `entity`** to
  **one or more named mesh objects** in the model via `object_id`, with a
  `type3d` that says how state is reflected:
  - `light` — toggle/illuminate a light object (with `lumens`/`decay`/`distance`),
  - `color` — recolor the object's material by state via `colorcondition`,
  - `hide`/`show` — visibility by state,
  - `text` — render the entity state as a sprite/label on the object,
  - plus doors/covers (`door`, `doortype`, `hinge`, `pane`, rotation/sliding),
    rotation (`rotate`), and gestures (`gesture`/`tap_action`).
- `object_id` may name a single mesh, **or** reference an **object group**
  written as `<group_name>`, expanded against the top-level `object_groups`
  list. `createObjectGroupConfigArray()` in `src/helpers.ts` does this expansion;
  `createConfigArray()` flattens the per-entity configs that
  `hasConfigOrEntitiesChanged()` then watches for HA state changes. These
  helpers are the heart of the mapping — read them first.
- Object names come from the 3D model itself (the modeler names the meshes), so
  the config is tightly coupled to a specific exported model file.

## Where things live

```
src/
├── floor3dpro-card.ts   # THE card LitElement: Three.js scene, model load,
│                        #   state→3D mapping, render scheduler, asset cache.
│                        #   @customElement('floor3dpro-card'); customCards reg.
├── editor.ts            # floor3dpro-card-editor LitElement (the visual config
│                        #   editor; transactional draft/commit "Pro" editor).
├── types.ts             # Floor3dCardConfig + EntityFloor3dCardConfig + Pro types.
├── const.ts             # CARD_VERSION (keep in sync with package.json).
├── helpers.ts           # config-array builders (the mapping expansion),
│                        #   mergeDeep, getLovelace, pro_skill/pro_log runtime.
├── ensureComponents.ts  # lazy-loads HA's hui-* / ha-* editor sub-elements.
└── localize/
    ├── localize.ts       # localStorage-driven string lookup
    └── languages/        # en.json, nb.json (translation tables)
elements/                 # MWC web-component wrappers, re-tagged floor3dpro-*
│                        #   (button/select/textfield/formfield) for isolation.
dist/floor3dpro-card.js   # BUILD OUTPUT — committed; the HACS-distributed bundle.
rollup.config.js          # production build (terser; outputs dir: 'dist', ES).
rollup.config.dev.js      # dev/watch build + serve on :5000.
rollup-ignore-plugin.js   # strips duplicate MWC side-effect imports from bundle.
hacs.json                 # HACS metadata (filename: floor3dpro-card.js).
.github/workflows/hacs.yml # CI: HACS validation only (no lint/build/test).
ENGINE.md                 # the "game-engine backbone" architecture write-up.
README.md                 # user-facing docs (rendered on the HACS page).
```

## Adding / changing a feature — walkthrough

Example: add a new `type3d` behavior or a new config option.

1. **Add the config field** to `Floor3dCardConfig` (or
   `EntityFloor3dCardConfig`) in `src/types.ts`. If it's a sensible default for
   new cards, also add it to `getStubConfig()` in `src/floor3dpro-card.ts`.
2. **Implement the behavior** in `src/floor3dpro-card.ts`: the binding is set up
   in/after `setConfig()` (which builds `_configArray` / `_object_ids` via the
   `src/helpers.ts` array builders), and state is applied in the per-entity
   update path that `hasConfigOrEntitiesChanged()` gates. For object-group
   support, route the `object_id` through the existing
   `createObjectGroupConfigArray()` expansion rather than parsing `<…>` yourself.
3. **Editor**: surface the option in `src/editor.ts` if it should be editable in
   the UI. The editor is transactional (draft → commit) — follow the existing
   `setConfig`/commit-scheduler pattern; don't auto-commit on every keystroke.
4. **Translations**: add any new user-facing strings to **both**
   `src/localize/languages/en.json` and `nb.json` (same key path).
5. **Bump the version** in both `package.json` and `src/const.ts`.
6. **Lint + build**: `npm run lint && npm run build`, then **commit the updated
   `dist/floor3dpro-card.js`**.

If you change the **card tag** or **public config options**, treat it as a
breaking contract change — see Ripple awareness.

## Ripple awareness

- **HACS = versioned public contract.** The custom element tag
  (`custom:floor3dpro-card`), the config option names/shapes in `types.ts`, and
  the bundle filename (`dist/floor3dpro-card.js`, per `hacs.json`) are all
  consumed by end users' dashboards via HACS releases. Renaming the tag or a
  config key, or changing a `type3d` semantic, breaks existing dashboards —
  bump the version and call it out in `README.md` / release notes. Don't rename
  the tag casually (it also exists to avoid colliding with upstream
  `floor3d-card`).
- **`ha-assets` consumes this card.** The sibling Home Assistant config repo
  (`../ha-assets`) uses `type: custom:floor3dpro-card` in
  `dashboards/dashboard-dashboard/floor-plan-v2.json`, loading the bundle from
  `path: /local/community/floor3dpro-card/` with a `.glb` model + `objectlist`
  and the `pro_log` / `pro_skill` opt-ins. A tag rename, a config-option
  rename/semantic change, or a changed resource path needs a matching update to
  that dashboard (and to the Lovelace `resources:` registration that serves the
  bundle).
- **Fork hygiene.** This is a fork of upstream `adizanni/floor3d-card`. Keep
  local "Pro" changes identifiable and rebase-friendly: they're marked with
  `Faz-*` / `pro` comments and isolated behind the `floor3dpro-` tag prefix and
  opt-in `pro_skill`/`pro_log` config (default off). Prefer additive, clearly
  marked changes over scattered edits to upstream logic, so future merges from
  upstream stay tractable. Per the README, this fork's scope is
  scaling/stabilization, not feature expansion of the original.
- **`dist/` drift.** The committed bundle is the artifact HACS ships. Forgetting
  to rebuild `dist/` after a source change ships stale behavior — always rebuild
  and commit it together.

## When stuck

- 3D scene / rendering / loaders / animation? → `src/floor3dpro-card.ts` (and
  `ENGINE.md` for the scheduler/asset-cache rationale).
- "How does an entity drive a 3D object?" → the mapping model section above, then
  the config-array builders in `src/helpers.ts` and `setConfig()`.
- Config options / shape? → `src/types.ts` (`Floor3dCardConfig`) +
  `getStubConfig()` for defaults.
- Editor UI behavior / draft-commit? → `src/editor.ts`.
- Build output / bundling / dev server? → `rollup.config.js` /
  `rollup.config.dev.js` + `rollup-ignore-plugin.js`.
- HACS / distribution / versioning? → `hacs.json`, `package.json` version +
  `src/const.ts`, `.github/workflows/hacs.yml`.
- Translations? → `src/localize/`.
- Upstream comparison? → `repository` field in `package.json` and the original
  `adizanni/floor3d-card` repo.
