# NukeEngine-Eco

A modular C++20 game engine by [Luastris](https://luastris.com): a core DLL with curated
exports, a host **editor**, a runtime **player** that ships games, and hot-pluggable
**modules** (renderer, physics, audio, scripting, runtime GUI) behind POD service seams.
This repository is the ecosystem ROOT — every part as a submodule plus the superbuild
that builds them all with one command. **Best starting point.**

## Disclaimer

And we're back! Extremely sorry for such a long silence — the engine and I ran into an
unresolvable pile of troubles, so we needed some time to wake up. For a long while I had
neither the time nor the brain to move NukeEngine forward, but now there are some
possibilities again. I can't promise I will always be able to support NukeEngine — but
you're welcome to contribute!

My little dream of a user-friendly, modular, extensible engine with built-in modding
support is growing up. Maybe one day it'll be ready to become the foundation of our
games ;)

## Ecosystem

| Repository | What it is |
|------------|------------|
| [NukeEngine-Eco](https://github.com/Luastris/NukeEngine-Eco) | **This repo** — everything as submodules + the superbuild + nukegen (reflection for C++ < 26). Best for a quick start. |
| [NukeEngine](https://github.com/Luastris/NukeEngine) | The core (engine DLL). |
| [NukeEngine-Editor](https://github.com/Luastris/NukeEngine-Editor) | The editor host. |
| [NukePlayer](https://github.com/Luastris/NukePlayer) | The player (the game exe your dist ships). |
| [NukeRenderDiligent](https://github.com/Luastris/NukeRenderDiligent) | **Required.** The main renderer (Diligent Engine: D3D11/D3D12, RT). |
| [NukeImGui](https://github.com/Luastris/NukeImGui) | **Required for the editor.** The editor's shared ImGui DLL — never ships with a game. |
| [NukeGUI](https://github.com/Luastris/NukeGUI) | Runtime (in-game) GUI module — optional if your game draws no GUI. |
| [NukePhysicsJolt](https://github.com/Luastris/NukePhysicsJolt) | Physics module (Jolt). |
| [NukeScript](https://github.com/Luastris/NukeScript) | Lua scripting backend. |
| [NukeCSharp](https://github.com/Luastris/NukeCSharp) | C# scripting backend (hosts modern .NET/CoreCLR). |
| [NukeAudio](https://github.com/Luastris/NukeAudio) | The default audio system. |
| [TestNUKEModule](https://github.com/Luastris/TestNUKEModule) | The pristine sample plugin (docs-in-code). Editor-only, auto-excluded from dists. |
| [NukeUtils](https://github.com/Luastris/NukeUtils) | Tools: nukegen (reflection codegen for C++ < 26, needs Python) + clean-release staging. |
| [NukeRenderBGFX](https://github.com/Luastris/NukeRenderBGFX) | Legacy bgfx renderer — WIP on pause, not required, not maintained. |
| [NukeRenderOGL](https://github.com/Luastris/NukeRenderOGL) | Legacy OpenGL renderer — not maintained. |

## Quick start

```
git clone --recurse-submodules https://github.com/Luastris/NukeEngine-Eco.git
cd NukeEngine-Eco
cmake -S . -B build -G "Visual Studio 17 2022" -A x64        # VCPKG_ROOT must be set
cmake --build build --config Debug -- /m
```

The **superbuild** drives the engine + editor solution and every module that is PRESENT
in the tree — a module you didn't pull is skipped with a notice, nothing breaks. Run dir
is `NukeEngine/x64/<Config>` (editor, player, `modules/`, `shaders/` all land there).
Requirements: VS2022 (v143) with the C++ workload (incl. ATL), [vcpkg](https://github.com/microsoft/vcpkg)
with `VCPKG_ROOT` set, Python on PATH (reflection codegen), the .NET SDK if you build
NukeCSharp.

## What's new (since the last public update)

The engine went through a total structural refactor — and came out better. Highlights,
grouped:

### Architecture & core
- Engine API lives in the `nuke::` namespace; renamed to the engine's own nomenclature:
  GameObject → **Atom**, Scene → **World**, NukeComponent → **Component**.
- C++20 (next stop — C++26 with native reflection).
- **Forward-compatible reflection** covering the full API, static members included
  (nukegen for C++ < 26); the inspector, serialization, scripting and modding all ride it.
- **Everything is a module** behind POD service seams: renderer, physics, audio,
  scripting, runtime GUI — hot-reload-friendly plugins with a unified export; swap
  renderers (Diligent / legacy bgfx / OGL) by config.
- Component ids (correct serialization + identification), quaternion math end to end
  (the editor shows euler angles as a stable VIEW of the quaternion).
- Jobs system (multithreading): core pinning — main on 0, physics on the last, the rest
  dynamic or configured; async imports and builds ride the pool.
- Config moved Lua → JSON; full dependency upgrade & cleanup (dead submodules removed).

### Rendering
- **New renderer: Diligent Engine** (D3D11/D3D12; RT reflections today, DLSS/FrameGen
  groundwork laid). bgfx moved out of the engine as a legacy module — WIP on pause.
- PBR: metallic-roughness materials, normal/specular/metal-rough/occlusion/emissive maps;
  lights + shadow mapping (dir/point/spot, PCF); per-pixel transparency with correct
  ordering; frustum culling.
- Environment: sky, sun, moon, stars, day cycle, moon phases; reflection probes with
  box-projection parallax.
- Post: MSAA + FXAA + TAA, bloom, SSR, color grade, vignette, custom post shaders.
- HDR10 output (player only, Windows).
- Texture pipeline: BC compression with import-time encoding choice (BC1 small/lossy,
  BC3 for diffuse+alpha, BC5 for normals), sampling modes, RenderTextures, GIF support.
- Render-pass object-id feature for shaders; window abilities via config (borderless,
  sizable, transparent, always-on-top), forced dark titlebar + per-window icon (Windows).

### Editor
- Total refactor on ImGui 1.92: dockable windows, interface icons, PIE (play-in-editor,
  WYSIWYG), persistent editor state, project structure.
- **Browser:** selection, creation, renaming (a renamed `.cs` keeps its class in sync),
  drag & drop, view modes (tiles/list/tree), file opening, forward/back navigation
  (M4/M5 by default); packed sessions list pak/mod content too.
- **Hierarchy** with drag & drop; save a prefab by dropping an atom onto the browser;
  prefab ↔ world instance sync both ways.
- Inspector for files + dedicated editors per file type (text/material/mesh/prefab/
  audio preview) in native OS windows; item/asset pickers everywhere (no raw text
  fields for enumerable values).
- Undo/redo action history; cascade resource deletion with link clearing; dirty
  detection + hierarchical diff/merge resolving for externally edited files.
- Gizmos + debug drawing (light directions, collider volumes, triggers, camera frusta),
  clickable entity icons in the viewport.
- Project Settings: plugins, default world, hotkey map (conflict-aware rebindable pool),
  packaging options; engine Preferences (external IDE detection — VS/Rider/VSCode/N++,
  opens files in the RUNNING instance at the exact line).
- Console: severity filters with live counts, text filter, copy, and double-click ANY
  line with a `path(line)` reference (compiler errors included) to jump to the source.
- Status bar + API for modules; crash telemetry (symbolized stack on any crash/assert).

### Scripting
- **Scripting is a shared service** — a game can run Lua AND C# side by side.
- Lua runtime → module (NukeScript), reflection-driven bindings (no hand-written
  wrappers), LuaBridge 2.0 → 3.0, hot reload.
- **.NET support (NukeCSharp):** hosts modern CoreCLR via hostfxr (net8.0 floor,
  RollForward), `Electron` base class, generated TYPED API from the reflection registry
  (components via `GetComponent<Light>()`, real enums, double-precision math primitives
  matching C++ 1:1 — `Vector2/3/4`, `Quaternion` with the full rotation toolkit, `Color`,
  `Math`), hot reload mid-PIE, compiled `GameScripts.dll` ships inside the game pak.
- **Full object model in every language:** every reflected Model class is first-class in
  C++, C# and Lua — create assets from script, look them up BY NAME (never guids), assign
  objects to objects, and push CONTENT: texture pixels, procedural mesh geometry, audio
  from memory.
- **Modding scripting:** a mod's C# compiles into its own assembly against the game's,
  loads ADDITIVELY (game classes stay), and ships inside the `.numod`.

### Physics, audio, animation
- Physics: Jolt module with multithreaded stepping, triggers, contact callbacks, ray/
  shape casts, editor debug volumes.
- Audio system: module (miniaudio, embedded decoders), plain ogg/wav/mp3/flac clips,
  spatial voices, buses, music analysis (beat/bass/chroma) feeding post effects.
- Animation system: skinned meshes, clips with events, serializable state machine +
  editor, retarget maps, FABRIK IK.

### Packaging & modding
- **Packaging:** File → Package Project builds a complete `dist/` — player exe renamed
  and icon-stamped, only the USED modules, cooked content in an immutable `game.nupak`
  (NUPAK container: store/zlib/zstd per entry, CRC-checked). Only the dependency closure
  of the manifest ships; shaders always ship (scripts bind them by name at runtime).
- **Modding:** mods are `.numod` overlays with dependencies (mods-on-mods), enabled via
  `config/mods.json`. Mods are POINT DIFFS: each records the world exactly as its author
  saw it, and the merge applies per-atom / per-component / **per-property** — a stale mod
  can never wipe what the base game gained since. Everything a mod added is BADGED with
  the mod's name in the editor.
- Opening a packed game (`.nupak`) mounts it read-only with a work overlay (Bethesda
  style — never extracted); opening a `.numod` mounts the base game + only that mod's
  dependency chain. The editor's mounted-mods list is SEPARATE from the game's.

### Build system & tooling
- **Superbuild** (this repo): one CMake tree drives the engine solution + every present
  module in dependency order, parallel via msbuild; absent modules are skipped.
- **The editor builds its own dependencies**: File → Build Engine, output streams into
  the Console, progress in the status bar, on a worker thread — and **Package Project
  always rebuilds Release first**, so a dist can never ship stale binaries.
- Extracted-module deploy fixes; every module deploys itself into the run dir post-build.

## Projects

A game is a **project**: a `*.nuproj` manifest (name, startup world, plugin list, service
choices, packaging settings) + a `content/` tree. The editor opens projects via
File → New Project / Open Project (`.nuproj`, or a packed `.nupak` / `.numod` — see Mods).
Switching projects relaunches the editor (the project lifecycle is pinned to boot).

## Packaging a game (File → Package Project)

Rebuilds Release through the superbuild FIRST, then produces a complete release under the
build path (default `<project>/dist`, configurable in Project Settings along with the
game name, icon and compression):

```
dist/
  <GameName>.exe        renamed NukePlayer.exe, game icon stamped into its resources
  *.dll                 engine + runtime deps (NukeImGui.dll excluded — editor-only)
  config/main.json      window title rewritten to the game name
  config/mods.json      mod list (created empty, user-editable)
  modules/              ONLY the modules the project uses; editor-only modules
                        (import NukeImGui.dll) are excluded automatically
  modules/managed/      the C# bridge (when NukeCSharp ships)
  mods/                 the mod socket (survives repackaging)
  content/game.nupak    the game: cooked content + manifest (canonical "game.nuproj")
                        + engine shaders/ and fonts/ INSIDE the pak
                        + managed/bin/GameScripts.dll (compiled C#)
```

Key mechanics:

- **Cooking:** only the dependency closure of the manifest ships (startup world → every
  asset it reaches). The editor walks ENGINE serialization only; every other file type is
  claimed by the module that owns it (`NUKEModule::cookContent`). Shaders (`*.hlsl`)
  always ship — scripts bind them by name at runtime. Dynamically composed paths are
  invisible to the cooker — list them in `"packInclude"` in the `.nuproj`. Modules name
  their extra shipping needs via `NUKEModule::shipExtras`.
- A RELEASE player refuses to run without a pak; dev builds can run the raw project tree.
- Packed content is served as **bytes only** — pak entries are never extracted to disk.
- C# players need the .NET Runtime (>= 8) installed; the SDK is only a dev-machine need.

## Mods

- **Format:** a mod is a `.numod` (same NUPAK container, store by default = editable)
  that overlays the game: same-path entries override, new entries add. Mounted above
  `content/game.nupak` in the order of `config/mods.json`. Engine shaders/fonts live in
  the pak, so mods can override those too. The console logs every override
  (`'path' OVERRIDDEN by mod: X`).
- **Point diffs:** Package Mod records each changed world's BASIS (the world exactly as
  the author saw it) inside the mod. At load, every layer diffs against its own basis and
  the merge applies per-atom / per-component / **per-property** — the mod imposes exactly
  what its author touched, no matter how the base game evolves afterwards. Everything a
  mod ADDS is badged `[modname]` in the hierarchy and the inspector.
- **Authoring:** open the game's `game.nupak` — it mounts read-only (never extracted),
  edits land in a `<stem>_mod` overlay beside it. Opening a `.numod` mounts the base game
  plus only THAT mod's dependency chain. The game's `config/mods.json` belongs to the
  PLAYER; the editor session's mounted mods are a separate list (the Mods panel has both
  columns).
- **File → Package Mod...** asks for the mod's NAME: each name is its own `.numod` (same
  name updates that mod), the diff is computed against the mounted stack, dependencies
  (`requires`) are recorded automatically, and the mod self-registers in the game's
  `config/mods.json`. A mod's compiled C# assembly ships inside it and loads additively.

## Dev hooks (env vars, fire ~2.5 s after editor boot)

| Variable | Effect |
|----------|--------|
| `NUKE_OPEN_PROJECT=<path>` | open a project (skipped if already open — the child inherits the env) |
| `NUKE_OPEN_WORLD=<rel>`    | open a world from project content |
| `NUKE_OPEN_ASSET=<rel>`    | open an asset editor |
| `NUKE_PACKAGE=1`           | run Package Project headlessly (builds Release first) |
| `NUKE_PACKAGE_MOD=1` or `=<name>` | run Package Mod headlessly (optionally named) |
| `NUKE_PLAY=1`              | enter PIE ~4 s after boot (headless play verification) |
| `NUKE_AUDIO_NULL=1`        | audio module in null-device mode (headless runs) |
| `NUKE_GPU_VALIDATION=1`    | Debug only: enable the D3D12 validation layer + DRED breadcrumbs (a device-removed / GPU-fault cause lands in the Console). Off by default — it costs FPS; turn it on only when the renderer is crashing. |

## Assorted gotchas (learned the hard way)

- Seam/vtable changes (`iRender`, `iAudio`, `iPhysics`, `NUKEModule`): append-only, at the
  END — then rebuild every module DLL in the same batch. Inserting mid-vtable silently
  corrupts stale modules.
- Debug = `/MDd`, Release = `/MD` — the two sets of binaries NEVER mix (CRT mismatch).
- The per-instance material block in world files is a HAND-ROLLED field list in
  `World.cpp` (SaveAtom/LoadAtom) — a new `Material` field must be added there too or it
  silently resets on world save / PIE stop / packaging.
- Renderer-internal shader pairs (ui/shadow/sky/post/debug/outline*) are excluded from
  the material-shader scan in ONE place: `RendererInternalShader()` in `resdb.cpp`.
- `dxcompiler.dll` + `dxil.dll` must sit next to the exe (vendored in
  `NukeRenderDiligent/deps/dxc`, deployed by its CMake post-build).
- `requires` is a C++20 keyword — the mod-dependency field is `requires_`.
