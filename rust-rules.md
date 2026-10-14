# Rust rules for the PhotoCraft codebase

A concise reference for style and practices used across this project. It is a distillation of
`AGENTS.md`, `docs/contributing.md`, `docs/architecture.md`, `clippy.toml`, `rustfmt.toml`, and the
workspace `Cargo.toml`. When these sources disagree, the source of truth is the file closest to the
code it governs (for example `clippy.toml` for clippy allowances, `rustfmt.toml` for formatting).

## 1. Language and target

- **Rust only.** No JavaScript or TypeScript. The desktop app is native egui/eframe on wgpu; the web
  build is the same Rust compiled to WebAssembly (trunk + wasm-bindgen). The web build may generate a
  small loader through `wasm-bindgen`, but never hand-write JS.
- **No Tauri, Electron, or any webview/JS UI framework** as a dependency, build step, or packaging target.
- **Edition 2024.** The workspace requires Rust 1.95.
- **License:** contributions are MIT OR Apache-2.0. The ArtCraft logos in `docs/brand/` are *not* open
  source (`docs/brand/LICENSE-brand.txt`).

## 2. Project identity

- Product name is written **PhotoCraft** (`{Function}Craft` PascalCase, as in its siblings ArtCraft,
  ArtCraftX, DesignCraft, VectorCraft, EffectCraft, FilmCraft, LightCraft, PdfCraft) in user-facing
  text: UI, window titles, About, installers, release names, docs prose.
- Machine names stay lowercase: crates (`photocraft-*`), binaries, file names, ids
  (for example `ai.storytold.photocraft`).

## 3. Workspace layout and layering

The workspace is a Cargo [workspace] of `crates/*`, `apps/*`, and `xtask`, with resolver "3".

**Layers (enforced by `cargo xtask layers`, checked in CI):**

```text
 L7  apps/*                         (binaries: wire everything together)
 L6  ui-egui · automation · platform
 L5  engine
 L4  tools · viewport · io · ml · plugins
 L3  compose · gpu · format
 L2  ops · algo · paint · text · vector
 L1  doc
 L0  geom · cms · color · raster     psd, codecs, raw, adobe-assets (standalone, no workspace deps)
```

**Layering rules:**

1. A crate may depend only on crates in *lower* layers. No cycles and no sideways dependencies except
   where explicitly listed.
2. Nothing below L6 may depend on any UI toolkit, winit, or a `platform` implementation. Platform
   services reach the engine through traits defined in `engine` (or in `platform`'s trait-only core)
   and are injected at startup.
3. `photocraft-psd` depends on *no* workspace crate. The doc↔PSD mapping lives in `io`. The same rule
   holds for every standalone crate, with one documented exception: `codecs` → `heif` (both standalone
   and publishable; `STANDALONE_EXCEPTIONS` in `xtask/src/layers.rs`).
4. **wasm gate:** every crate in L0–L5 (except feature-gated native backends) must build for
   `wasm32-unknown-unknown`. CI runs
   `cargo build -p photocraft-engine --target wasm32-unknown-unknown --no-default-features --features web`.
5. `gpu` is optional for `engine`. Engine features are `gpu` (default on) and `cpu-only`; `cpu-only`
   builds are what the headless CLI and CI tests use.
6. **C dependencies** (optional LibRaw, pdfium) only behind features, only in `codecs`, `raw` or `io`,
   and never on by default for the web target.

**Current built crates:** `geom`, `cms`, `color`, `raster`, `psd`, `codecs`, `doc`, `ops`, `algo`,
`paint`, `text`, `vector`, `compose`, `gpu`, `format`, `raw`, `io`, `plugins`, `engine`, `ui-egui`,
`automation`, `testkit`, plus the three apps.

**Not yet split out:** `viewport` and `tools` live inside `ui-egui` and `engine`; `platform` services
are function hooks injected by each app (`ui_egui::Services`); `adobe-assets` and `ml` are not started.

**Crate granularity:** split `algo` into `-adjust`, `-filters`, `-select`, `-inpaint` and `-warp` once
any module passes about 10k lines, or once compile times hurt. Internal module boundaries should already
follow those lines.

**Naming:** directories are short and packages are prefixed `photocraft-`. `photocraft-psd` gets a
neutral, publishable name (for example `psd-rw`) if released to crates.io.

## 4. Never crash (outranks feature work)

People trust PhotoCraft with their work; a crash loses it. A malformed file, a bad command or MCP param,
a corrupt settings file, an odd keystroke, or a full disk must produce an error the user or agent can act
on, never a panic. Do not ship a feature by adding a panic path; fix a crash before building on top of it.
The shared standard is `../craftrules/standards/never-crash.md`.

### 4.1 Non-test code never panics

No `unwrap()`, `expect()`, `panic!`, `unreachable!`, `todo!`, or `unimplemented!` in non-test code. Return
the crate's error type and propagate with `?`. Prefer `ok_or(..)?`, `let .. else { return Err(..) }`,
`if let`, or `unwrap_or*` where a fallback is truly correct (never one that silently corrupts a document).
Unfinished features return an "unsupported" error.

The only exception is a *provably infallible literal*: `#[allow(clippy::expect_used)]` plus
`.expect("why it can't fail")`.

### 4.2 No `unsafe`

The workspace sets `unsafe_code = "forbid"`. The one exception is the isolated helper crate
`photocraft-tablet` (`crates/tablet`): winit drops pen tablet data and reading it on macOS needs an
AppKit event monitor (Objective-C interop). Only its `src/macos.rs` allows `unsafe`
(`unsafe_code = "deny"` crate-wide, every block has a `SAFETY:` comment, tested against real `NSEvent`s);
its X11 path and all mapping code are safe. Do not add `unsafe` anywhere else.

### 4.3 Input-derived numbers are hostile

- Use `get()` rather than `[i]`/`[a..b]` for indices from files, params, selections, or arithmetic on
  them.
- Slice strings only at char boundaries.
- Use `checked_*`/`saturating_*` for lengths, offsets, and counts.
- Guard division by zero and NaN/inf casts.
- Cap allocations sized by input.

### 4.4 Bound recursion

Bound recursion with depth limits or seen-sets. Documents can be deep or cyclic.

### 4.5 Don't cascade

Handle lock poisoning (`lock().unwrap_or_else(PoisonError::into_inner)`) and treat thread joins as
`Result`s.

### 4.6 Last-resort guard

The app shell must catch an escaped panic around command dispatch and file import/export, report it as an
error, and keep the document. It is a safety net, not a licence to panic. Keep `panic = "unwind"`.

### 4.7 Prove it

Every crash fix comes with a small synthetic regression test that panicked before the fix.

### 4.8 Enforced by clippy

`clippy.toml` allows `unwrap`/`expect`/`panic`/indexing in tests only:

```toml
allow-unwrap-in-tests = true
allow-expect-in-tests = true
allow-panic-in-tests = true
allow-indexing-slicing-in-tests = true
```

Clean crates carry:

```rust
#![deny(clippy::unwrap_used, clippy::expect_used, clippy::panic, clippy::unimplemented, clippy::todo, clippy::unreachable)]
```

New crates start with that attribute.

### 4.9 Commands must never panic on input

A command's `run` closure and everything it calls must return `Err`, never panic, for *any* params or
document state: validate params, check bounds before indexing or dividing, and reject absurd sizes before
allocating. The `panic_hunt` integration test fuzzes every command with adversarial params and must stay
green. New commands come with a graceful-failure test (empty/out-of-range/wrong-type params → `Err`, not a
crash).

## 5. Design principles

1. **Engine-first, headless-first.** Every feature is reachable without a GUI: from tests, the CLI, and
   MCP. The GUI is one client of the engine.
2. **Everything is a command.** Each user-visible action has a stable `CommandId` (for example
   `filter.blur.gaussianBlur`) with typed, serializable parameters. The menu, command palette, shortcuts,
   recorded actions/macros, CLI, MCP, and plugins all dispatch the same commands.
3. **Data describes the UI; toolkits draw it.** Tools produce *overlay primitives* (lines, handles,
   marching-ants paths) as data. Filter and adjustment dialogs are generated from *parameter schemas*. A
   new toolkit implements one renderer for each, not about 150 bespoke dialogs.
4. **One algorithm, one parameter struct.** Every algorithm has a CPU reference implementation (Rust +
   rayon, deterministic). Some also get a GPU implementation (WGSL). Both read the same `#[repr(C)]`
   `bytemuck::Pod` parameter struct. Never keep hand-synced copies of an algorithm in several languages.
5. **Immutable snapshots, copy-on-write tiles.** Pixel data is stored in `Arc`-shared 256² tiles. Taking a
   document snapshot is O(layers), which makes undo, background jobs, autosave, and UI reads cheap and
   lock-free.
6. **Determinism.** The same input gives the same output across CPU and GPU (within tolerance), across
   thread counts, and between preview and export. Noise is hashed from document coordinates.
7. **Pure Rust by default.** C/C++ dependencies are allowed only behind Cargo features, in I/O-edge crates
   (for example an optional LibRaw).
8. **Clean-room.** Study other editors' *behaviour* only, and **do not copy proprietary source** (Rust,
   WGSL, C++, or JS), even where it ships as readable source. Specs come from public format docs (Adobe PSD
   spec, ISO/ICC), academic papers (PatchMatch, Poisson blending, ARAP), and observed behaviour. Never paste
   code from another product.

### 5.1 Avoiding GIMP's hole: bake data, not assumptions

The root cause of GIMP's historical holes is that fundamental properties were **baked into the core as
assumptions instead of being data** (pixel depth, colour model, "a layer is a pixel buffer"). PhotoCraft
avoids this from day one with the following rules; the test is that **adding 64-bit float, a new colour
model, or a new file format should touch a leaf crate, not the foundation.**

| Property | Rule | Where |
|---|---|---|
| **Bit depth** | Every surface carries a runtime `PixelFormat { depth: U8 \| U16 \| F16 \| F32, model, alpha }`. Algorithms are written generically over a `Sample` trait and monomorphized per depth, or run on f32 working buffers and convert at tile edges. No `u8` pixel type appears in any public engine API. CI runs the golden suite at 8, 16, and 32f. | `raster`, `color`, `algo` |
| **Colour model** | `ColorMode` (RGB, Gray, CMYK, Lab, Indexed, Bitmap, plus multichannel/spot) is in `doc` from the start. Channel count is dynamic (1–N plus alpha plus spot). Only RGB/Gray *rendering* ships first, but the storage, PSD round-trip, and conversion paths never assume 3 or 4 channels. | `doc`, `raster`, `psd` |
| **Colour management** | Every document has an ICC profile. The compositor takes a working-space descriptor. The display transform is a separate final stage. Blending space (gamma vs linear) is a per-document option. | `color`, `compose` |
| **Non-destructive by construction** | `LayerContent` is an enum that includes Adjustment, Fill, Text, Shape, and Smart from v1, with smart filters as a stack on any layer. The compositor evaluates a *graph*, not a list of flattened buffers. Destructive ops are just "apply and bake". | `doc`, `compose` |
| **File formats** | Formats plug in through `trait ImageDecoder/Encoder` plus a registry with capability flags (depths, modes, layers, alpha, metadata, ICC). Adding a format is one crate feature. It must never need a change in core types. Lossless PSD round-trip preserves unknown blocks. | `codecs`, `io`, `psd` |
| **Size** | Tiled, sparse storage behind a `TileStore` trait, with 64-bit coordinates for PSB-scale canvases (300k × 300k). Nothing assumes a document fits in one allocation or one GPU texture. | `raster` |
| **UI toolkit** | The core never depends on the toolkit. The UI crate can be replaced without touching the engine. | layering rule 2 |

## 6. Formatting and linting

- **Format with `cargo fmt`.** Rustfmt is configured for wide lines and compact small items (same style
  as the other craft apps: EffectCraft, FilmCraft):

  ```toml
  # rustfmt.toml
  max_width = 160
  use_small_heuristics = "Max"
  ```

- **Lint with `cargo clippy -- -D warnings`.** The workspace lints are:

  ```toml
  [workspace.lints.rust]
  unsafe_code = "forbid"

  [workspace.lints.clippy]
  dbg_macro = "warn"
  todo = "warn"
  ```

- Match surrounding code. Comments explain *why*.

## 7. Crates, features, and dependencies

- **Internal crates** are workspace path dependencies prefixed `photocraft-` (for example
  `photocraft-geom`, `photocraft-cms`, `photocraft-engine`). The workspace also pins shared external
  dependencies (for example `thiserror = "2"`, `serde = { version = "1", features = ["derive", "rc"] }`,
  `arc-swap = "1"`, `bytemuck = { version = "1", features = ["derive"] }`, `rayon = "1"`, `wgpu`, `kurbo`,
  `parley`, `skrifa`, `vello_cpu`, `rmcp`, and so on) in `[workspace.dependencies]`.
- **Profiles:**
  - `[profile.dev.package."*"]` sets `opt-level = 2` because image code is unusable at opt-level 0.
  - `[profile.dev]` uses `opt-level = 1`.
  - `[profile.release]` uses `lto = "thin"`, `codegen-units = 1`.
  - `[profile.wasm-release]` exists for the browser build: fat LTO, opt-level "s" for most crates, but
    `opt-level = 3` for the pixel-processing crates (`raster`, `color`, `cms`, `compose`, `algo`, `paint`)
    so filters and compositing stay fast; it strips debuginfo. The browser build is capped at 24 MiB
    (packaging/web/package.sh fails above it); the wasm profile took the wasm from 26.3 MiB to 18.8 MiB.
  - `panic` stays the target default for wasm32-unknown-unknown (it aborts); `panic = "unwind"` is for the
    native app shell.

## 8. Commands (the engine API)

**New user-visible behaviour = a command in the engine** (`crates/engine/src/*_cmds.rs`, registered in
`commands.rs`) with id, label, menu path, shortcut, params doc, `enabled`, and `run`, plus tests. Put new
commands in a **new module** (`engine/src/<area>_cmds.rs` with a `specs()` function) rather than growing a
shared file. The UI, CLI, control channel, and MCP all dispatch commands by id.

- Use the **exact id from `crates/ui-egui/src/menu_catalog.rs`** and the menu item goes live automatically.
- Commands without a Photoshop menu entry use a descriptive id in the same style (for example
  `layer.smartFilter.delete`) and an empty menu path.
- Only pure view/window state (zoom, panels, screen mode) belongs to the shell (`menus.rs`
  `UI_COMMANDS`).

### 8.1 Command shape

Each command registers:

- `id` (`"filter.blur.gaussianBlur"`), `label`, `menu_path` (`["Filter","Blur"]`), and a
  `default_shortcut`.
- `params: schemars::Schema`, plus optional `ui_hints` (slider ranges, units, log scale, grouping).
- `enabled(&DocSnapshot, &ToolState) -> bool` and `checked(...) -> Option<bool>`.
- `preview: PreviewMode` (None / Live / Proxy). This lets dialogs show on-canvas preview generically.
- `kind`: Instant / Dialog / Interactive (hands control to a tool, e.g. Free Transform).

Params docs look like `{"radius":px=4,"mode":"a|b"="a"}` — that is what agents read through
`command_list`.

### 8.2 Adding a command (checklist)

1. Find the id: search `crates/ui-egui/src/menu_catalog.rs` for the Photoshop menu item.
2. Algorithm goes in the lowest crate that fits (`algo` for imaging, `paint`, `vector`, `text`, `cms`),
   with unit tests. It takes depth-agnostic surfaces (`photocraft-raster`), works per tile, and is
   deterministic (seeded randomness).
3. Command goes in an engine module (`crates/engine/src/<area>_cmds.rs`) exposing `specs()`, registered
   with `v.extend(...)` in `commands.rs`. Fill in:
   - `id`, `label`, `menu` path, and Photoshop's default `shortcut`;
   - a params doc string such as `{"radius":px=4,"mode":"a|b"="a"}`;
   - an `enabled` predicate (greys the menu item out);
   - `run`, which returns a JSON result and records exactly **one** history step (use `coalesce` for drags
     and typing sessions).
4. Respect the context: active selection (feathered), layer vs mask target, locks, the colour model, and
   depth.
5. Tests in the module: behaviour, undo/redo, disabled states, bad params, several depths.
6. Dialog (if it has parameters): filters and adjustments get schema-driven dialogs with live preview
   (`ui-egui/src/filter_dialog.rs`, `dialogs.rs`).
7. Run `cargo xtask parity` and commit the updated `docs/parity.md`.

## 9. Formatting, colour, and pixel code

- **No format or colour assumptions.** Bit depth (8/16/32f) and colour model (RGB/Gray/CMYK/Lab…) are
  runtime data. Never introduce a `u8`-only pixel path in public APIs. Never assume sRGB: colour conversions
  go through `photocraft-cms` (`Transform`, `transform::cached`). Test at several depths.
- **Algorithms** are generic over a `Sample` trait and monomorphized per depth, or run on f32 working
  buffers and convert at tile edges.
- **Every algorithm:** signature is a pure function
  `fn(input: &SurfaceView, params: &P, region: Rect, cancel: &CancelToken) -> Surface`. Params:
  `P: Pod + JsonSchema + Serialize + Deserialize`. Registration is done by the engine, not by the algo crate
  itself, so `algo` stays UI- and engine-agnostic. Tile-aware, requesting an input halo of `radius` pixels.
  Test with golden-image tests in `testkit`, plus benchmarks with criterion.

### 9.1 Algorithms inventory

| Area | First-pass contents | GPU kernel? |
|---|---|---|
| Adjustments | levels, curves, brightness/contrast, exposure, vibrance, hue/sat, color balance, B&W, photo filter, channel mixer, gradient map, selective color, invert/posterize/threshold, 3D LUT | yes (LUT + small kernels) |
| Filters | gaussian/box/motion/radial/surface/lens blur, unsharp/smart sharpen, high pass, noise add/reduce/median/dust, distort family (twirl, wave, polar, spherize, displace), stylize, render (clouds, lens flare), pixelate | many |
| Selection | marquee/lasso geometry → mask, magic wand (flood fill tolerance), color range, quick select (graph cut / superpixels), feather, grow/shrink, refine edge | some |
| Inpaint/heal | spot heal, healing brush (Poisson), content-aware fill (PatchMatch + multiscale EM) | later |
| Warp | free transform (affine/perspective/warp mesh), liquify, puppet (ARAP), lens correction, resampling (Lanczos/bicubic) | yes |
| Paint (`paint`) | dab generation (hardness, roundness, angle, spacing, scatter), dynamics from pressure/tilt/velocity, smoothing (pulled string / lazy mouse), wet edges, airbrush, eraser, clone/heal sources, blend modes per stroke, `.abr` import | CPU first, GPU later |
| Text (`text`) | font DB trait (system fonts native, bundled fonts on web), parley layout, per-run styles, paragraph options, warp; renders to a raster cache | — |
| Vector (`vector`) | shapes, bezier paths, pen tool geometry, strokes (dash, align, caps), boolean ops → coverage via vello_cpu | — |
| ML (`ml`) | `InferenceBackend` trait; tasks: `segment_point/box` (SAM2), `matte_subject` (background removal), `depth`, `sky`; model registry with SHA-256 verification and on-demand download | backend-dependent |

## 10. UI, theme, and visual verification

- **The UI is thin and data-driven.** UI state lives in `ui-egui/src/state.rs` (serde), so the control
  channel can read and drive it. Colours and radii come from `theme::Tokens`, never hard-coded. Use
  `theme::Tokens` and `widgets::*`.
- **Verify UI changes visually.** Render offscreen with
  `cargo run -p photocraft-ui-egui --example snapshot` (no window, no focus stealing), or launch with
  `--control` and take `ui.screenshot`. Look at the PNG. Demo images must be public-domain art, never
  personal photos.
- When fetching assets, never put a person's name, email, or other personal details in requests
  (User-Agent, headers, URLs); use a generic `Photocraft-dev` User-Agent.

## 11. Performance is a feature

Benchmark heavy operations on a 24–36 MP image in release. Work per tile in parallel (rayon), skip empty
tiles, never scan a full surface per frame (cache per revision), and record before/after timings in the dev
log.

**Performance targets for v1:**

- Stroke-to-pixel within one frame at 120 Hz.
- Pan and zoom at display refresh on a 100 MP / 30-layer document.
- Open a 100 MP JPEG in under 1.5 s.
- 20k×20k 16-bit documents on a 16 GB machine.

## 12. WASM

**Never break wasm.** L0–L6 must `cargo check --target wasm32-unknown-unknown` (run `cargo xtask wasm`).
File-system code is `cfg(not(target_arch = "wasm32"))` or goes through the platform services. C
dependencies are never on by default for the web target.

## 13. PSD and file formats

- **`photocraft-psd`** is a standalone, publishable PSD/PSB (v1/v2, 64-bit lengths) crate with **no
  dependency on the workspace document model.** The doc↔PSD mapping lives in `io`. The merged composite on
  write is supplied by the caller as an image buffer; `io` renders it with `compose`, so `psd` never needs a
  compositor.
- **Lossless round-trip:** unknown or unsupported tagged blocks are kept as raw bytes and re-emitted
  unchanged, so files keep features not yet modelled (for example 3D or video).
- **Lazy decode:** the parser indexes channel data and decodes it only on request, so opening a 2 GB PSB is
  fast and memory-bounded.
- **Mapping fidelity levels** (in `io::psd`): flattened image → raster layers, masks, blend/opacity/fill,
  visibility, names → groups (including pass-through), clipping, blend-if, adjustment and fill layers → layer
  effects (lfx2) → text layers (descriptors kept for round-trip) → smart objects (embedded and linked, live
  both ways) → vector masks and shape layers.
- **Standalone format crates:** `psd`, `codecs`, `raw`, `heif`, `adobe-assets` depend on nothing in the
  workspace and are publishable. `codecs` → `heif` is the one documented standalone exception.
- **Native format (`photocraft-format`):** a `.pcraft` file is a zip (store mode) or directory bundle:
  `manifest.json` (versioned document tree with serde, `format_version`, migrations),
  `tiles/<blake3>.zst` (content-addressed, zstd-compressed), `thumb.png` and `composite/` mips. Saving is
  incremental (only new tiles plus the manifest; garbage-collect unreferenced ones). Autosave and crash
  recovery reuse the same writer into a recovery directory, reading from a snapshot on a background thread.
  PSD is the interchange format; `.pcraft` is the lossless native one.

### 13.1 Keeping the native format complete

`photocraft-format` deliberately fails to compile when a `photocraft-doc` struct gains a field, so nothing
is silently dropped from `.pcraft` saves. When you add a doc field, add it to `crates/format/src/manifest.rs`
and `convert.rs` with `#[serde(default)]` so older files still load. If the field has a PSD equivalent, map it
in `crates/io` too, and keep unknown PSD blocks verbatim.

## 14. Documentation, assets, and attribution

- **Read these first:** `docs/architecture.md` (crate map, engine/UI seam, document model),
  `docs/development.md` (build, test, run, drive the app programmatically, debug tricks),
  `docs/contributing.md` (rules; the "add a command" checklist), `docs/control-protocol.md` (JSON control
  channel), `docs/ui-design.md` (design tokens, themes, widgets, how to match Photoshop's look),
  `docs/roadmap.md` (honest parity assessment, milestones, current focus), `docs/parity.md` (every Photoshop
  menu item, live or missing), `docs/scorecard.md` (generated scorecard), and `crates/<name>/README.md`
  where present.
- **Clean-room, redux:** third-party assets must be permissively licensed, keep their license file next to
  them, and get a row in `ATTRIBUTION.md` (path, title, author, source, license) in the same change; so do
  original assets.
- **Never commit font files.** Fonts live in [storytold/craft-fonts](https://github.com/storytold/craft-fonts)
  (rules: `../craftrules/standards/fonts.md`). PhotoCraft uses them through the optional build input
  `CRAFT_FONTS_DIR=<craft-fonts checkout>` (read by `crates/text/build.rs`, exposed as
  `photocraft_text::CRAFT_FONTS`; empty without it, and everything must still work). Build with it:
  `CRAFT_FONTS_DIR="$PWD/../craft-fonts" cargo build`.
- **Reusable learnings go in `../craftrules`**, never code; repos don't share code.

## 15. Testing

- **Tests are the gate.** Every change comes with tests. Format crates use round-trip, synthetic-generator,
  oracle, and fuzz tests. Keep the PSD corpus results and the parity floor
  (`crates/ui-egui/src/parity.rs`) from regressing.
- **Per-layer testing:**
  - Unit: per crate; algorithms against small hand-computed fixtures.
  - Golden images: `testkit::assert_image_eq(actual, "golden/name.png", tol)`; goldens regenerated by
    `xtask bless`.
  - CPU/GPU parity: every GPU kernel vs its CPU reference, software adapter in CI (≤1/255 for 8-bit output).
  - PSD: round-trip, composite oracle, fuzzing.
  - Engine: command-level tests — open fixture → dispatch commands → assert snapshot/render; action replays.
  - UI: headless egui tests (egui_kittest) for panels; kept thin because the UI is thin.
  - Perf: criterion benches + budget checks in CI (composite 4k² 20 layers, brush dab throughput, open 100
    MP).
  - Portability: CI matrix: macOS arm64, Windows x64, Linux x64, wasm32 build check.
- **Pixel code is tested at 8, 16, and 32-bit.**
- **Test corpora.** Real-file corpora live in `corpus/` (gitignored, never committed), fetched at pinned
  commits and sha256-verified by `cargo xtask corpus --all`: Photoshop-authored oracles from
  https://github.com/storytold/photocraft-corpus plus psd-tools, ag-psd, and PngSuite from their upstreams.
  Pins: `xtask/src/corpus_pins.rs`. Corpus tests are opt-in (cargo feature `corpus`): plain `cargo test`
  skips them, and with the feature on a missing corpus fails ("run `cargo xtask corpus --all`").
  `cargo xtask test-corpus` fetches and runs them all. CI always runs them (the `corpus` job, cached by pin).
  Never commit corpus files; new oracles go to photocraft-corpus (its `AGENTS.md`), then a pin bump here.

## 16. Automation, plugins, and extensibility

- **Everything is a command** means CLI and MCP parity come for free: `photocraft-cli run in.psd --cmd
  filter.blur.gaussianBlur --params '{"radius":4}' --out out.png`.
- **`automation`**: an MCP server built on `rmcp`. Tool names use underscores (dotted names such as
  `doc.open` belong to the JSON control protocol, `docs/control-protocol.md`). It exposes
  `session_list`, `doc_open`, `doc_new`, `doc_save`, `doc_export`, `doc_inspect` (layer tree as JSON),
  `doc_render_preview` (PNG), `doc_select`, `doc_close`; `command_list`, `command_run`, `command_batch`;
  `jobs_list`, `jobs_cancel`; and a bridge-only set for a running app (`ui_inspect`, `ui_screenshot`,
  `ui_pointer`, `ui_menu_invoke`, `ui_set`, `control_call`). Stdio for agent CLIs, optionally loopback TCP
  with a token to attach to a running GUI. `AuthorizedWorkspace` holds independent read and write directory
  capabilities; remote paths are validated relative names, and engine commands that still require ambient
  filesystem access fail closed at the automation boundary.
- **Actions:** recorded `Vec<CommandInvocation>`, replayable in batch (File → Automate → Batch).
- **Plugins:** sandboxed WebAssembly filter plug-ins (`photocraft-plugins`, L4) run by `wasmi`, a
  pure-Rust interpreter, with fuel, memory, stack, and wall-time limits and no host imports. They are driven
  by the `plugin.*` commands. Native Photoshop `.8BF`/CEP/UXP hosting is out of scope (it needs unsafe FFI
  and can't run on the web). Panel plug-ins are later.
- **Scripting (later):** embed a scripting language over the same registry. Options are Rhai, or Lua via
  mlua (C). JS via QuickJS is possible if Photoshop-script familiarity is wanted.

## 17. Commit and PR hygiene

- **Commits:** small, focused, with a clear subject line.
- **Before you finish a task:**

  ```sh
  cargo test -p <crates you touched>
  cargo clippy -p <crates> --all-targets -- -D warnings
  cargo xtask layers
  cargo xtask wasm            # if you touched L0–L6
  cargo xtask parity          # if you added commands; commit the regenerated docs/parity.md
  cargo test -p photocraft-engine --test panic_hunt -- --ignored   # if you added/changed commands
  cargo xtask scorecard       # if you moved a number; commit the regenerated docs/scorecard.md
  cargo xtask perf --quick    # if you touched a hot path
  cargo xtask test-corpus     # if you touched psd, io, codecs, compose, gpu, text, or format
  ```

- **Parallel agents:** use your own target dir (`CARGO_TARGET_DIR=target/agent-<name>`) to avoid the Cargo
  build lock, and edit only the files you own. Shared files (`engine/src/lib.rs`, the `v.extend(...)` list in
  `engine/src/commands.rs`, `ui-egui/src/menus.rs`, `state.rs`) get small, surgical edits; re-read before
  editing. Keep every `Cargo.toml` valid at all times: the `crates/*` glob means one broken manifest breaks
  everyone's build. Create or rewrite manifests atomically: write to a temp file outside `crates/`, then `mv`
  it into place. Disk space: each target dir is about 10 GB; delete `target/agent-*` dirs of finished agents.
- Then append a terse entry to `log/devlog.md` (what landed, numbers, what's still open). Sessions can end
  abruptly (crashes, context limits), so the dev log plus a green tree is how the next agent picks up.

## 18. Where things are tracked

- `docs/roadmap.md`: milestones M0–M12, status, and the current focus.
- `docs/parity.md`: generated Photoshop menu coverage.
- `docs/scorecard.md`: generated scorecard (sources: `scorecard/*.toml`, `perf/budgets.toml`,
  `perf/baseline.json`, corpus floors, prefs audit).
- `docs/releasing.md`: cutting a release (`cargo xtask version`, the `release` branch), signing secrets,
  packaging scripts in `packaging/`.
- `../craftrules/release/playbook.md`: how every storytold app builds signed release binaries (the canonical
  recipe; `docs/release-playbook.md` just points there); `docs/releasing.md` is PhotoCraft's specifics.
- `plan/` (local, gitignored): research, parity plan, execution plan, estimates.
- `log/` (local, gitignored): the dev log.

## 19. Contributor credits (About window)

- About ▸ Contributors/Models are compiled into the binary from `contributors/contributors.json`
  (commit stats; generated, never hand-edit) and `contributors/people.toml` (names people chose for
  themselves). See `docs/contributors.md`.
- **Agents working for a contributor:** when preparing a PR, check whether the human's GitHub username has a
  `[people.<username>]` entry in `contributors/people.toml`. If not, ask them once whether they want to be
  credited by more than their username (real name, display name, and/or public GitHub profile name with
  `sync_github_name = true`). If yes, add **only their own** entry (copy the template at the top of the file,
  or run `python3 ../../craftrules/scripts/contributors.py --add-me . --real-name "…" --sync-github-name`)
  and include it in the PR, committed as them. If no, change nothing: they are credited as `@username`
  anyway.
- Never add, edit, guess, or copy anyone else's entry or name (not from git config, commit authors, or GitHub
  profiles). Never hand-edit `contributors.json`.
- Maintainers refresh the stats with `python3 ../../craftrules/scripts/contributors.py .` (it also
  re-verifies who wrote each `people.toml` entry; `--check` only verifies).

## 20. Quick-reference cheat sheet

| Rule | What it means in code |
|---|---|
| Never crash | No `unwrap`/`expect`/`panic!`/`todo!`/`unimplemented!`/`unreachable!` in non-test code; `Result` + `?` everywhere |
| No `unsafe` | `unsafe_code = "forbid"`; only `crates/tablet/src/macos.rs` allows it, with `SAFETY:` comments |
| Hostile input | `get()` not `[i]`; char-boundary string slicing; `checked_*`/`saturating_*`; guard div-by-zero, NaN/inf; cap input-sized allocations |
| Bounded recursion | depth limits or seen-sets |
| No lock cascades | `lock().unwrap_or_else(PoisonError::into_inner)`; treat joins as `Result` |
| Commands, not handlers | new features go in `engine/src/<area>_cmds.rs` with `specs()`, registered via `v.extend(...)` in `commands.rs` |
| One algorithm, one param struct | CPU ref + optional WGSL share `#[repr(C)] bytemuck::Pod` params |
| No u8 pixel path in public APIs | generic over `Sample`, monomorphized per depth, or f32 working buffers |
| No sRGB assumption | convert through `photocraft-cms` |
| UI is thin | state in `ui-egui/src/state.rs` (serde); colours/radii from `theme::Tokens` |
| Wasm-safe | L0–L6 build `wasm32-unknown-unknown`; fs code is `cfg(not(target_arch = "wasm32"))` or platform services |
| Layer discipline | crate may only depend on lower layers; `cargo xtask layers` must pass |
| Standalone crates are publishable | `psd`, `codecs`, `raw`, `heif`, `adobe-assets` depend on nothing in the workspace |
| Clean-room | no copying proprietary code/shaders/profiles/assets; specs from public docs and papers |
| Tests are the gate | round-trip, golden, oracle, fuzz, CPU/GPU parity, `panic_hunt`; pixel code at 8/16/32-bit |
| Performance is a feature | per-tile parallel (rayon), skip empty tiles, cache per revision, benchmark on 24–36 MP |
| Format/lint | `cargo fmt` (max_width 160, use_small_heuristics "Max"), `cargo clippy -- -D warnings` |
