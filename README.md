# NukeEngine-Eco

A modular C++20 game engine by [Luastris](https://luastris.com): a core DLL with curated
exports, a host **editor**, a runtime **player** that ships games, and hot-pluggable
**modules** (renderer, physics, audio, scripting, runtime GUI) behind POD service seams.
This repository is the ecosystem ROOT — every part as a submodule plus the superbuild
that builds them all with one command. **Best starting point.**

> **Scope in one line:** NukeEngine is built for **single-player desktop games with deep
> modding** — **Windows x64 + macOS + Linux (Deuterium), no mobile, no consoles**
> (see [Scope & platforms](#scope--platforms)).

## Disclaimer

And we're back! Extremely sorry for such a long silence — the engine and I ran into an
unresolvable pile of troubles, so we needed some time to wake up. For a long while I had
neither the time nor the brain to move NukeEngine forward, but now there are some
possibilities again. I can't promise I will always be able to support NukeEngine — but
you're welcome to contribute!

My little dream of a user-friendly, modular, extensible engine with built-in modding
support is growing up. Maybe one day it'll be ready to become the foundation of our
games ;)

## Scope & platforms

NukeEngine was designed from day one as a **modular engine for single-player desktop games
with a broad modding toolset** — and the architecture is shaped by exactly that: a host with
hot-pluggable modules, reflection-driven tooling, per-property mod diffs, and a mod pipeline
built INTO the editor instead of bolted on afterwards. Everything below follows from that
choice.

| Target | Where we stand |
|--------|----------------|
| **Platform** | **Windows 10/11 x64**, **macOS (Apple Silicon / Intel)** and **Linux x64** — the Deuterium cross-platform milestone. |
| **Linux specifics** | Native Vulkan incl. **hardware ray tracing** (vendored DXC), X11 + native Wayland (switchable), the editor ships as a self-contained **AppImage** and packages games into AppImages. |
| **Mobile (iOS / Android)** | **Not supported and not planned** for the foreseeable future. |
| **Consoles** | **Not supported and not planned** for the foreseeable future. |
| **Focus** | Single-player games. |
| **Modding** | First-class: packed games mount read-only, mods are point diffs, mod C# loads additively. |

**Why mobile/console are a no:** it's a matter of principle, not capability. Every hour spent
on GLES/Metal ports, touch UX, console SDKs and certification is an hour not spent making the
desktop experience deeper — so we don't spend it. The engine is aggressively optimized for
desktop multicore (the job system pins and saturates every core; scenes run at many hundreds
of FPS) and takes full advantage of desktop-class GPUs — hardware ray tracing, heavy world
systems — because that's the hardware its games target. Mobile and consoles aren't a gap
waiting to be filled; they're outside the mission.

## Ecosystem

| Repository | What it is |
|------------|------------|
| [NukeEngine-Eco](https://github.com/Luastris/NukeEngine-Eco) | **This repo** — everything as submodules + the superbuild + nukegen (reflection for C++ < 26). Best for a quick start. |
| [NukeEngine](https://github.com/Luastris/NukeEngine) | The core (engine DLL). |
| [NukeEngine-Editor](https://github.com/Luastris/NukeEngine-Editor) | The editor host. |
| [NukePlayer](https://github.com/Luastris/NukePlayer) | The player (the game exe your dist ships). |
| [NukeRenderDiligent](https://github.com/Luastris/NukeRenderDiligent) | **Required.** The main renderer (Diligent Engine: **Vulkan** / D3D12 / D3D11, hardware RT on Vk and D3D12). |
| [NukeImGui](https://github.com/Luastris/NukeImGui) | **Required for the editor.** The editor's shared ImGui DLL — never ships with a game. |
| [NukeGUI](https://github.com/Luastris/NukeGUI) | Runtime (in-game) GUI module — optional if your game draws no GUI. |
| [NukePhysicsJolt](https://github.com/Luastris/NukePhysicsJolt) | Physics module (Jolt). |
| [NukeScript](https://github.com/Luastris/NukeScript) | Lua scripting backend. |
| [NukeCSharp](https://github.com/Luastris/NukeCSharp) | C# scripting backend (hosts modern .NET/CoreCLR). |
| [NukeAudio](https://github.com/Luastris/NukeAudio) | The default audio system. |
| [NukeVFX](https://github.com/Luastris/NukeVFX) | Particle VFX: emitters, shapes, forces, sub-emitters, trails, mesh particles, RT-visible particles. |
| [NukeWater](https://github.com/Luastris/NukeWater) | Water: ocean/lake/river/waterfall surfaces, buoyancy, flow, spread, FLIP volumes, caustics. |
| [NukeTilemap](https://github.com/Luastris/NukeTilemap) | Grid worlds (colony sims / roguelikes): tilesets, layers, `.nutile`. |
| [NukeTilemapEditor](https://github.com/Luastris/NukeTilemapEditor) | Editor companion for `.nutile` — editor-only, never ships with a game. |
| [NukeAtlasImporter](https://github.com/Luastris/NukeAtlasImporter) | `.atlas` (libgdx / TexturePacker) importer — editor-only, never ships. |
| [NukeGamepad](https://github.com/Luastris/NukeGamepad) | Gamepad input provider (GLFW) — a module by design, like every other peripheral. |
| [TestNUKEModule](https://github.com/Luastris/TestNUKEModule) | The pristine sample plugin (docs-in-code). Editor-only, auto-excluded from dists. |
| [NukeUtils](https://github.com/Luastris/NukeUtils) | Tools: nukegen (reflection codegen for C++ < 26, needs Python) + clean-release staging. |
| [NukeRenderBGFX](https://github.com/Luastris/NukeRenderBGFX) | Legacy bgfx renderer — WIP on pause, not required, not maintained. |
| [NukeRenderOGL](https://github.com/Luastris/NukeRenderOGL) | Legacy OpenGL renderer — not maintained. |

## Quick start (Windows)

```
git clone --recurse-submodules https://github.com/Luastris/NukeEngine-Eco.git
cd NukeEngine-Eco
cmake -S . -B build -G "Visual Studio 17 2022" -A x64        # VCPKG_ROOT must be set
cmake --build build --config Debug -- /m
```

The **superbuild** drives the engine + editor solution and every module that is PRESENT
in the tree — a module you didn't pull is skipped with a notice, nothing breaks. Run dir
is `NukeEngine/x64/<Config>` (editor, player, `modules/`, `shaders/` all land there).
Requirements: **Windows 10/11 x64**, VS2022 (v143) with the C++ workload (incl. ATL),
[vcpkg](https://github.com/microsoft/vcpkg) with `VCPKG_ROOT` set, Python on PATH (reflection
codegen), the .NET SDK if you build NukeCSharp, and a GPU with up-to-date **Vulkan** drivers
(the editor's default backend; hardware RT needs an RT-capable GPU).

## Quick start (macOS)

One-time machine setup (Xcode Command Line Tools assumed):

```
brew install cmake pkg-config vulkan-loader molten-vk        # Vulkan runtime = MoltenVK
git clone https://github.com/microsoft/vcpkg && ./vcpkg/bootstrap-vcpkg.sh
export VCPKG_ROOT=$PWD/vcpkg
```

Check out the tree + the two vendored deps that are NOT git submodules (gitignored on
purpose — the Windows dev tree carries them the same way):

```
git clone --recurse-submodules https://github.com/Luastris/NukeEngine-Eco.git
cd NukeEngine-Eco
git clone https://github.com/DiligentGraphics/DiligentCore.git \
    NukeRenderDiligent/deps/DiligentEngine/DiligentCore
git -C NukeRenderDiligent/deps/DiligentEngine/DiligentCore \
    checkout $(grep -o '`[0-9a-f]\{40\}`' NukeRenderDiligent/patches/README.md | tr -d '\`')
git -C NukeRenderDiligent/deps/DiligentEngine/DiligentCore submodule update --init --recursive
git clone https://github.com/kunitoki/LuaBridge3.git NukeScript/deps/LuaBridge3
git -C NukeScript/deps/LuaBridge3 \
    checkout $(git -C NukeScript ls-tree HEAD deps/LuaBridge3 | awk '{print $3}')
```

(The lua sources themselves are vendored directly inside NukeScript — nothing extra to fetch.)

Engine dependencies (classic vcpkg install, one long first run) + the ONE shared dynamic
GLFW every window-touching module loads (three static copies would collide in objc). The
default build is **universal (arm64 + x86_64)** — the triplets live in
`vcpkg-overlays/triplets/`:

```
"$VCPKG_ROOT/vcpkg" install assimp boost-atomic boost-bind boost-chrono boost-config \
    boost-container boost-dll boost-filesystem boost-function boost-smart-ptr boost-system \
    boost-thread boost-tokenizer boost-tuple glfw3 glm lua meshoptimizer nlohmann-json \
    stb zstd zlib --triplet=universal-osx --overlay-triplets=vcpkg-overlays/triplets
"$VCPKG_ROOT/vcpkg" install "glfw3:universal-osx-dynamic" --overlay-triplets=vcpkg-overlays/triplets
```

(For fast native-only iteration: configure with `-DNUKE_MACOS_UNIVERSAL=OFF` and install
the same list for `arm64-osx` / `x64-osx` instead.)

Build (single-config trees; the editor's File → Build Engine drives the same dirs):

```
cmake -S . -B build-mac -DCMAKE_BUILD_TYPE=Debug             # VCPKG_ROOT must be set
cmake --build build-mac -j10
cmake -S . -B build-mac-release -DCMAKE_BUILD_TYPE=Release   # Package Project needs it
cmake --build build-mac-release -j10
```

Every binary is stamped with its architecture set: the configure banner, the boot log
(`[config] build: arm64+x86_64 (running arm64)`) and the bundles' `NukeBuildArchitectures`
plist key all say the same thing — no guessing. `stage_release.sh` prints a `lipo` report.

Run dir is `NukeEngine/macos/<Config>`; the editor is `NukeEngine-Editor.app` inside it,
and every build's final step mirrors the runtime (player, dylibs incl. MoltenVK, modules,
shaders, fonts) into the bundle — the `.app` is **self-contained**: copy it alone to
`/Applications` and it runs, edits and packages games. In the dev tree the loose run-dir
layout beside the `.app` always wins (fresh binaries; the inner copies are only loaded by
a bundle standing alone). Package Project emits a self-contained `<Game>.app`. Per-machine
state (imgui layout, saved config, shader caches, saves) lives in
`~/Library/Application Support/<App>/` for installed bundles — never beside or inside a
deployed `.app`; dev trees keep everything in the run dir, same as Windows. The .NET SDK
is needed only for NukeCSharp. macOS differences: rendering is **Vulkan via MoltenVK**
(hardware RT is unavailable — MoltenVK exposes no `VK_KHR_ray_tracing_pipeline`; the
engine falls back to raster exactly like on non-RT GPUs), HDR output rides the ST2084
swap chain (EDR), and window transparency uses the alpha-composited swap chain instead
of DirectComposition.

GUI launches (Finder/Dock) don't see your shell profile: the editor appends the standard
toolchain homes to `PATH` (Homebrew, `/usr/local`, the .NET and CMake.app dirs) and
discovers `VCPKG_ROOT` on its own (build-time location first, then `~/vcpkg`,
`~/projects/vcpkg`, `/opt/vcpkg`, ...) — both logged at boot, an env var still wins.
Game-module builds (File → Build & Reload Game Modules) reuse a classic vcpkg install
when one has the engine's triplets (`universal-osx`/`arm64-osx`/`x64-osx`; on Windows
`x64-windows`); without one, the vcpkg manifest builds the public deps into the module's
build dir on first configure — slow once, and `libbacktrace` needs
`brew install autoconf autoconf-archive automake libtool`.

## Quick start (Linux)

One-time machine setup — a C++20 toolchain, the windowing/audio dev headers and vcpkg:

```
# Debian/Ubuntu
sudo apt install build-essential cmake ninja-build git curl zip unzip tar pkg-config \
    autoconf autoconf-archive automake libtool bison flex \
    libx11-dev libxrandr-dev libxinerama-dev libxcursor-dev libxi-dev libxext-dev \
    libxfixes-dev libxkbcommon-dev libwayland-dev wayland-protocols \
    libgl1-mesa-dev libegl1-mesa-dev libvulkan-dev \
    libasound2-dev libpulse-dev libudev-dev libdbus-1-dev
# Fedora
sudo dnf install gcc-c++ cmake ninja-build git curl zip unzip tar pkgconf-pkg-config \
    autoconf autoconf-archive automake libtool bison flex \
    libX11-devel libXrandr-devel libXinerama-devel libXcursor-devel libXi-devel \
    libXext-devel libXfixes-devel libxkbcommon-devel wayland-devel wayland-protocols-devel \
    mesa-libGL-devel mesa-libEGL-devel vulkan-loader-devel \
    alsa-lib-devel pulseaudio-libs-devel systemd-devel dbus-devel

git clone https://github.com/microsoft/vcpkg ~/vcpkg && ~/vcpkg/bootstrap-vcpkg.sh
export VCPKG_ROOT=$HOME/vcpkg
```

Optional but recommended: `dotnet-sdk-8.0` (NukeCSharp), `zenity` **or** `kdialog`
(native file dialogs; the editor picks whichever matches your desktop).

Then the whole thing is one script — it clones the two vendored deps that are NOT git
submodules (DiligentCore at its pinned commit, LuaBridge3 — gitignored on purpose, the
Windows dev tree carries them the same way), runs the classic vcpkg install (one long
first run) including the ONE shared dynamic GLFW built with **both** X11 and Wayland
backends, then configures and builds:

```
git clone --recurse-submodules https://github.com/Luastris/NukeEngine-Eco.git
cd NukeEngine-Eco
NukeUtils/build_linux.sh --deps                 # first time: deps + Debug build
NukeUtils/build_linux.sh --release              # Package Project needs the Release tree
NukeUtils/build_linux.sh --appimage             # + squash NukeEngine-Editor.AppImage
```

(`--deps` is idempotent; later iterations are just `NukeUtils/build_linux.sh`. The same
`build-linux`/`build-linux-release` trees are what the editor's File → Build Engine
drives. The Linux DXC — SM6.x shaders, ray tracing — vendors itself at configure time,
pinned by hash, same arrangement as the Windows `deps/dxc` DLLs.)

Run dir is `NukeEngine/linux/<Config>`; the editor binary sits there, and every build's
final step mirrors the runtime (player, .so's incl. DXC and the shared GLFW, modules,
shaders, fonts, stock config) into **`NukeEngine-Editor.AppDir`** — a flat, self-contained
image: run it in place via `./AppRun`, or squash it into a single-file
**`NukeEngine-Editor.AppImage`** (`--appimage`, or
`NukeUtils/bundle_editor_linux.sh <rundir> --appimage <out>`). Package Project emits a
`<Game>.AppDir` **and** a ready `<Game>.AppImage` — the editor carries `appimagetool`
inside its own image, so an installed editor needs nothing from the host. Per-machine
state (imgui layout, saved config, shader caches, saves) lives in `~/.config/<App>/` for
installed images — an AppImage mounts read-only; dev trees keep everything in the run
dir, same as Windows.

Linux differences: rendering is **native Vulkan** — including **hardware ray tracing**
on RT-capable GPUs (the vendored `libdxcompiler.so` compiles the SM6.5 RayQuery shaders;
without it the engine falls back to raster). Display server: **games run native Wayland**
by default (X11 fallback); **the editor runs on X11/XWayland** by default — its
multi-window UX (tear-off panels, drag-to-dock) needs client-side window positioning and
a global cursor, which the Wayland protocol forbids. The choice is explicit in
Preferences → Interface → **Display server** (with the trade-offs spelled out in its
tooltip), `NUKE_DISPLAY_BACKEND=x11|wayland` overrides everything; on native Wayland
detached windows keep a dedicated caption button that re-docks them. `.nuproj`
double-click association registers via XDG mime (Preferences → System). GUI launches see
the same PATH/VCPKG_ROOT discovery as on macOS (`~/.local/bin`, the distro dotnet homes,
snap/flatpak exports are probed and logged at boot).

## Technologies

### Render backends — the editor runs on Vulkan

One HLSL shader source set, three backends (`0` = D3D11, `1` = D3D12, `2` = Vulkan):

| Backend | Used as | What it brings |
|---------|---------|----------------|
| **Vulkan** (`2`) | **editor default; the only backend on macOS (MoltenVK)** | Native ImGui multi-viewport: panels and asset editors detach into real per-window swapchains with full dock previews. Hardware ray tracing (`VK_KHR_ray_tracing_pipeline`, opted in at device creation; not exposed by MoltenVK — raster fallback there), tessellation, background shader compilation. HLSL goes through glslang, RT shaders (SM6.x) through the vendored DXC emitting SPIR-V, plus our own SPIR-V disk cache (`config/shadercache_vk/`) so warm boots match D3D12. On macOS the swap chain also carries HDR (ST2084/EDR) and alpha-composited transparency. |
| **Direct3D 12** (`1`) | **packaged-game default on Windows** (a macOS config asking for D3D is forced to Vulkan with a log line) | Hardware ray tracing, DirectComposition transparent windows, HDR10 output (player only). Detached editor windows fall back to the GDI-blit host path. |
| **Direct3D 11** (`0`) | legacy fallback | No ray tracing. For old hardware / driver triage only. |

> ⚠️ **Don't switch the editor off Vulkan.** `Preferences → Editor Render Backend` (an
> engine-wide preference in `%APPDATA%` / `~/Library/Application Support`, applied on the next editor restart) exists mostly for
> triage. Vulkan is the backend the editor is developed and tested on: its WSI multi-window
> path is what the whole detachable-window architecture rides on, while the D3D route goes
> through DXGI secondary swapchains and the GDI-blit host fallback — historically a minefield.
> Running the editor on D3D11/D3D12 risks instability and crashes. The **runtime** backend is a
> separate, project-level setting (`Project Settings → Render Backend` → `config/main.json`
> `window.backend`), and there **D3D12 is the tested default** — RT reflections, DComp
> transparency and HDR10 live there.

### Stack

| Technology | Where it's used | Upstream license |
|------------|-----------------|------------------|
| **C++20** (MSVC v143) — next stop C++26 native reflection | engine, editor, all modules | — |
| **HLSL** (single shader source for every backend) | renderer + module shaders | — |
| **Diligent Engine** (DiligentCore: Vulkan / D3D12 / D3D11) | NukeRenderDiligent | Apache-2.0 |
| **Vulkan / SPIR-V toolchain** — Vulkan-Headers, volk, glslang, SPIRV-Tools, SPIRV-Cross | via DiligentCore | Apache-2.0 / MIT / BSD (per component) |
| **DirectX Shader Compiler** — one vendored `dxcompiler.dll` (+ `dxil.dll`) serves both backends: DXIL for D3D12, SPIR-V for Vulkan | SM6.x + ray-tracing shaders | University of Illinois/NCSA (LLVM) |
| **Dear ImGui 1.92** + **ImGuizmo** | NukeImGui (editor UI, gizmos) | MIT |
| **Jolt Physics** | NukePhysicsJolt | MIT |
| **miniaudio** (embedded decoders: ogg/wav/mp3/flac) | NukeAudio | public domain / MIT-0 |
| **Lua 5.5** + **LuaBridge3** | NukeScript | MIT |
| **.NET 8+ / CoreCLR** via `hostfxr` | NukeCSharp (players need the runtime, not the SDK) | MIT |
| **Assimp** | mesh / model / animation import → native formats | BSD-3-Clause |
| **Boost** (dll, filesystem, thread, chrono, container, …) | engine core (plugin loader, FS, time) | BSL-1.0 |
| **GLFW** (patched: `WS_EX_NOREDIRECTIONBITMAP` for DComp transparency) | windowing, gamepad | Zlib |
| **glm** · **nlohmann::json** · **stb** · **zstd** / **zlib** | math · serialization + config · images · NUPAK compression | MIT · MIT · MIT + public domain · BSD-3-Clause / Zlib |
| **Python 3** | nukegen reflection codegen — build-time only, never shipped | PSF |
| **CMake ≥ 3.20** + msbuild, **vcpkg** | superbuild, dependency acquisition | — |
| **bgfx**, **OpenGL** | legacy renderers, not maintained | BSD-2-Clause / — |

Third-party components stay under **their own licenses** — Luastris claims no rights in them.
The authoritative copy of each license travels with the component in its own tree (`deps/`,
vendored directory, or the `vcpkg` manifest that fetches it); the license identifiers above are
a convenience summary, not a substitute. See `LICENSE.md` §16, and keep their notices in
anything you ship.

**Name & trademarks.** NukeEngine is an independent project and is **not** affiliated with,
sponsored by, or endorsed by The Foundry Visionmongers Ltd. or its *Nuke* compositing software,
nor with any other product whose name contains "Nuke". The name comes from this engine's own
atomic nomenclature — a World is built out of **Atoms**, and releases are codenamed after the
chemical elements in atomic-number order, by their LATIN names (Hydrogenium, Helium, …;
isotopes mark major revisions of the same release — Deuterium is Hydrogenium gone
cross-platform) — with its own logo, its own concept and its own niche. All other trademarks
belong to their owners (`LICENSE.md` §11).

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
- **New renderer: Diligent Engine** (Vulkan / D3D12 / D3D11; RT reflections today, DLSS/FrameGen
  groundwork laid). bgfx moved out of the engine as a legacy module — WIP on pause.
- **Vulkan backend** — the editor's default: native ImGui multi-viewport (real detachable
  windows), hardware ray tracing (RT shaders compiled to SPIR-V by the same vendored DXC), own
  SPIR-V disk cache. Packaged games stay on D3D12 (DComp transparency + HDR10).
  See [Technologies](#technologies).
- PBR: metallic-roughness materials, normal/specular/metal-rough/occlusion/emissive maps;
  lights + shadow mapping (dir/point/spot, PCF); per-pixel transparency with correct
  ordering; frustum culling.
- Environment: sky, sun, moon, stars, day cycle, moon phases; reflection probes with
  box-projection parallax.
- Post: MSAA + FXAA + TAA, bloom, SSR, color grade, vignette, custom post shaders.
- HDR10 output (player only; DXGI color space on Windows, ST2084/EDR swap chain on macOS).
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
- **Linux joins Deuterium**: the same superbuild configures with plain CMake/Ninja
  (`NukeUtils/build_linux.sh` wraps deps + configure + build + AppImage), the editor and
  packaged games ship as self-contained AppImages, DXC vendors itself for SM6.x/RT.
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

macOS games are **download-and-run**: the bundle carries its own universal Vulkan runtime
(`libMoltenVK.dylib`, official Khronos release — auto-vendored at configure time) plus the
shared `libglfw`, so a clean machine with no Vulkan SDK/brew needs nothing installed. Dev
machines still prefer the system Vulkan loader (validation layers) — the bundled MoltenVK
is the fallback.

On **macOS** the same layout lands inside a self-contained `dist/<GameName>.app`
(`Contents/MacOS/` holds the renamed player binary, `libNukeEngine.dylib`, `modules/`,
`config/`, `content/game.nupak`, `mods/`; the icon becomes `Resources/game.icns` via
sips/iconutil — the stock engine logo when the project sets none). Editor-only modules are
excluded by their `editorTool()` flag as well as the NukeImGui-import scan. At runtime the
bundle is never written to: per-user state goes to `~/Library/Application Support/<GameName>/`
(the user's `config/mods.json` there overrides the shipped one).

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
- **Native C++ modules in mods/DLCs:** a mod may ship engine plugins under `modules/`
  inside its pak. Machine code cannot run out of an archive (the OS loader needs a real
  file), so mounting caches them to `config/modcache/<mod>/` (byte-identical files are
  left untouched) and they are discovered — and enabled — alongside the host's own
  modules, with the same ABI gate. The game's plugin list cannot know a mod's modules;
  the mod being enabled IS the consent. A mod can never shadow a host module by reusing
  its file name (the host's `modules/` is discovered first).
- **Module dependencies:** `mod.json`/`pak.json` carry `"modules": ["NukeWater", ...]` —
  the engine plugins the content is built on. Package Mod/DLC detects the list (component
  types in packed JSON map to their owning plugin, native DLLs state their imports, script
  backends answer for their own files); a hand-written list in `mod.json` is
  authoritative and survives repacking. At mount, a mod/DLC whose module is missing is
  SKIPPED with a console line (its components would load as dead placeholders otherwise);
  a module the pak itself ships satisfies its own requirement.
- **Patching another mod's component (`"replaces"`):** a fix mod does not need the broken
  mod's sources. Prop-level bugs: override values with an ordinary point-diff mod.
  Behavior bugs: every compiled component exposes its props, `[[nuke::func]]` methods and
  `enabled` to ANY script through reflection — a fix script can mute the broken component
  and drive a replacement beside it. Terminal cases: put
  `"replaces": {"BuggyThing": "FixedThing"}` in the fix mod's `mod.json` — worlds and
  prefabs that saved `BuggyThing` construct `FixedThing` instead. Props load BY NAME
  through reflection, so a prop-compatible replacement picks up all saved data (its extra
  props default). Substitutions stack across mods (mount order wins, chains resolve up to
  4 hops); a replacement that is missing or disabled is ignored, so a disabled fix mod
  never breaks loading. Late plugin enables honor the map too — a placeholder upgrading
  after load cannot resurrect a replaced type.

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
- (Windows) `dxcompiler.dll` + `dxil.dll` must sit next to the exe (vendored in
  `NukeRenderDiligent/deps/dxc`, deployed by its CMake post-build). Not D3D-only: the same DLL
  compiles the SM6.x ray-tracing shaders for BOTH backends (DXIL for D3D12, SPIR-V for Vulkan),
  which is why Diligent is pointed at it instead of its default `spv_dxcompiler.dll`.
- `requires` is a C++20 keyword — the mod-dependency field is `requires_`.

## License

NukeEngine is **source-available** under the [NukeEngine License](LICENSE.md) (v1.1) — open to
read, use and modify, but **not** OSI "open source": redistribution of the engine itself is
restricted, and there are terms about what it may be used for. Read it. The short version:

- **Free to make games with, commercial included.** Ship your game and its player however you
  like; credit the engine, and say so if you built it with a *modified* engine/editor.
- **Don't republish the engine as a fork.** Contribute improvements back (encouraged) or keep
  them internal — never a rival build. Shipping a modified engine/editor to third parties needs
  a separate written license.
- **Your game's content is your business** — the license doesn't police themes; ratings and
  legal compliance are the developer's job. What's banned is **real-world harm** (CSAM, real
  incitement, deepfakes of real people, malware, fraud, unlawful surveillance) — modified
  copies and private internal builds included; no paid license buys an exception.
- **Sell your game however you like — just never sell chance.** Fixed-price DLC, skins,
  subscriptions, microtransactions where the buyer gets exactly what they paid for: all fine.
  Banned: real-money gambling and paid loot boxes/gacha. Randomness a player *earns by
  playing* is expressly fine — paid = deterministic, random = earned.
- **Plugins are yours to sell** — but a mod or plugin made for **someone else's game** may not
  be monetized without that developer's express permission, especially when their game is
  **free**. Giving it away is always allowed.
- **Don't use the editor to rip other people's games** — the game's rightsholder can enforce
  that clause directly.
- **Third-party components keep their own licenses** and we claim nothing in them
  (see [Technologies](#technologies) and `LICENSE.md` §16).

Contributions are accepted under the [CLA](CLA.md). Both documents state the author's intent in
plain terms and have **not** been reviewed by a lawyer.
