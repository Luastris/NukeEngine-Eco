# NukeEngine SUPERBUILD — one modular build for the whole tree.
#
#   cmake -S . -B build -G "Visual Studio 17 2022" -A x64     (VCPKG_ROOT must be set)
#   cmake --build build --config Debug            (or Release; -- /m for parallel msbuild)
#
# The ENGINE + EDITOR stay hand-maintained .vcxproj (nukegen prebuild, vcpkg autolink,
# dist deploy all live there) and build through their own NukeEngine.sln — driven here as
# a target so $(SolutionDir)-relative output paths (x64/<config>) stay EXACTLY the same.
# Every module/host is OPTIONAL: present -> builds, absent -> skipped with a notice
# (someone who didn't pull a module loses nothing). Modules deploy themselves into
# NukeEngine/x64/<config>/modules via their own POST_BUILD steps, unchanged.
cmake_minimum_required(VERSION 3.20)

# macOS: UNIVERSAL (arm64 + x86_64) by default — one build runs on every Mac, no guessing.
# -DNUKE_MACOS_UNIVERSAL=OFF drops to the host's native arch (halves compile time for quick
# iteration). Must be decided BEFORE project().
if(APPLE AND NOT DEFINED NUKE_MACOS_UNIVERSAL)
    set(NUKE_MACOS_UNIVERSAL ON CACHE BOOL "Build macOS binaries as arm64+x86_64 universal")
endif()
if(APPLE AND NUKE_MACOS_UNIVERSAL AND NOT DEFINED CMAKE_OSX_ARCHITECTURES)
    set(CMAKE_OSX_ARCHITECTURES "arm64;x86_64" CACHE STRING "macOS build architectures")
endif()

project(NukeAll NONE)

# Non-Windows (Deuterium): dependencies come from vcpkg's INSTALLED tree via plain
# CMAKE_PREFIX_PATH — deliberately NOT the vcpkg toolchain file: its add_library override
# breaks abseil (FetchContent'd by DiligentCore's SPIRV-Tools) with "links to itself".
# The vcpkg-exported *Config.cmake files carry debug/release locations themselves.
# Remember the vcpkg root in the cache: `cmake --build` re-configures WITHOUT the caller's
# environment, and losing VCPKG_ROOT mid-flight would silently drop every dependency path.
if(NOT WIN32 AND DEFINED ENV{VCPKG_ROOT} AND NOT "$ENV{VCPKG_ROOT}" STREQUAL "")
    set(NUKE_VCPKG_ROOT "$ENV{VCPKG_ROOT}" CACHE PATH "vcpkg root (captured at first configure)")
endif()
if(NOT WIN32 AND DEFINED NUKE_VCPKG_ROOT)
    if(NOT DEFINED NUKE_VCPKG_TRIPLET)
        if(APPLE)
            if(NUKE_MACOS_UNIVERSAL)
                set(NUKE_VCPKG_TRIPLET "universal-osx")   # overlay triplet (vcpkg-overlays/triplets)
            elseif(CMAKE_HOST_SYSTEM_PROCESSOR STREQUAL "arm64")
                set(NUKE_VCPKG_TRIPLET "arm64-osx")
            else()
                set(NUKE_VCPKG_TRIPLET "x64-osx")
            endif()
        else()
            set(NUKE_VCPKG_TRIPLET "x64-linux")
        endif()
    endif()
    if(EXISTS "${NUKE_VCPKG_ROOT}/installed/${NUKE_VCPKG_TRIPLET}")
        list(APPEND CMAKE_PREFIX_PATH "${NUKE_VCPKG_ROOT}/installed/${NUKE_VCPKG_TRIPLET}")
        # Ports that ship a Find*.cmake (stb for assimp's find_dependency(Stb MODULE)) put it
        # in their share/<port> dir — the toolchain would add these; we do it by hand.
        file(GLOB _nuke_vcpkg_findmods "${NUKE_VCPKG_ROOT}/installed/${NUKE_VCPKG_TRIPLET}/share/*/Find*.cmake")
        foreach(_fm ${_nuke_vcpkg_findmods})
            get_filename_component(_fmdir "${_fm}" DIRECTORY)
            list(APPEND CMAKE_MODULE_PATH "${_fmdir}")
        endforeach()
        list(REMOVE_DUPLICATES CMAKE_MODULE_PATH)
        message(STATUS "vcpkg installed tree: ${NUKE_VCPKG_ROOT}/installed/${NUKE_VCPKG_TRIPLET}")
    else()
        message(WARNING "vcpkg installed tree for ${NUKE_VCPKG_TRIPLET} not found under VCPKG_ROOT")
    endif()
    # One SHARED GLFW for every module that touches the window/input (renderer, ImGui,
    # gamepad) — three static copies each register their own Objective-C GLFW* classes and
    # the objc runtime warns about "mysterious crashes". Mirrors the Windows glfw3.dll
    # arrangement. Install once: `vcpkg install "glfw3:${NUKE_VCPKG_TRIPLET}-dynamic"`.
    set(_nuke_glfw_dyn "${NUKE_VCPKG_ROOT}/installed/${NUKE_VCPKG_TRIPLET}-dynamic")
    if(APPLE AND EXISTS "${_nuke_glfw_dyn}/lib/libglfw.3.dylib")
        set(NUKE_GLFW_DYLIB   "${_nuke_glfw_dyn}/lib/libglfw.3.dylib")
        set(NUKE_GLFW_INCLUDE "${_nuke_glfw_dyn}/include")
        set(NUKE_GLFW_LIBDIR  "${_nuke_glfw_dyn}/lib")
        message(STATUS "shared GLFW: ${NUKE_GLFW_DYLIB}")
    elseif(NOT APPLE AND EXISTS "${_nuke_glfw_dyn}/lib/libglfw.so.3")
        # Linux: same one-GLFW-per-process contract; the ELF soname libglfw.so.3 is what
        # DT_NEEDED records when modules link the path directly.
        set(NUKE_GLFW_DYLIB   "${_nuke_glfw_dyn}/lib/libglfw.so.3")
        set(NUKE_GLFW_INCLUDE "${_nuke_glfw_dyn}/include")
        set(NUKE_GLFW_LIBDIR  "${_nuke_glfw_dyn}/lib")
        message(STATUS "shared GLFW: ${NUKE_GLFW_DYLIB}")
    else()
        message(STATUS "shared GLFW not found (${_nuke_glfw_dyn}) — modules fall back to static glfw3 (objc class duplication on macOS; duplicated GLFW state elsewhere!)")
    endif()
endif()

if(WIN32 AND NOT CMAKE_GENERATOR MATCHES "Visual Studio")
    message(FATAL_ERROR "On Windows the superbuild drives .vcxproj projects — use a Visual Studio generator (-G \"Visual Studio 17 2022\" -A x64)")
endif()
if(NOT DEFINED ENV{VCPKG_ROOT})
    message(WARNING "VCPKG_ROOT is not set — module builds resolve vcpkg includes/libs through it")
endif()

# ---- GLFW transparency patch (WS_EX_NOREDIRECTIONBITMAP for DirectComposition) -----------
# The renderer's transparent-window support (Game.SetTransparent) needs GLFW to create its
# window WITHOUT a redirection bitmap, so a DirectComposition swap chain shows the desktop
# through alpha<1 pixels instead of an opaque white surface. GLFW upstream doesn't set that
# (it would break GL windows); our overlay port (vcpkg-overlays/glfw3) carries the one-line
# win32-only patch. Install glfw3 through the overlay at CONFIGURE time so the classic vcpkg
# install the modules consume already carries the patch — no manual step on a fresh checkout.
# Windows-only: the patch lives in win32_window.c; other platforms use GLFW's native
# transparent framebuffer and never see it. vcpkg no-ops fast when already patched.
if(WIN32 AND DEFINED ENV{VCPKG_ROOT} AND EXISTS "$ENV{VCPKG_ROOT}/vcpkg.exe")
    message(STATUS "GLFW transparency: ensuring patched glfw3 via overlay port")
    execute_process(
        COMMAND "$ENV{VCPKG_ROOT}/vcpkg.exe" install glfw3:x64-windows
                "--overlay-ports=${CMAKE_CURRENT_SOURCE_DIR}/vcpkg-overlays"
        RESULT_VARIABLE _nuke_glfw_rc
        OUTPUT_VARIABLE _nuke_glfw_log ERROR_VARIABLE _nuke_glfw_log)
    if(_nuke_glfw_rc EQUAL 0)
        message(STATUS "GLFW transparency: patched glfw3 ready")
    else()
        message(WARNING "GLFW overlay install failed (rc=${_nuke_glfw_rc}):\n${_nuke_glfw_log}\n"
            "Transparent windows may show an opaque surface until glfw3 is (re)installed with "
            "--overlay-ports=${CMAKE_CURRENT_SOURCE_DIR}/vcpkg-overlays")
    endif()
endif()

enable_language(C CXX)

# Run-dir subfolder + host-tool suffix, defined ONCE for every subproject (each module's
# CMakeLists also includes this itself for standalone configures).
include("${CMAKE_CURRENT_SOURCE_DIR}/NukeEngine/cmake/NukeRunDir.cmake")

# The build's architecture set, stamped into every binary (boot log, bundle plists) and
# shouted at configure time — nobody should ever have to GUESS what a build is.
if(APPLE)
    if(CMAKE_OSX_ARCHITECTURES)
        string(REPLACE ";" "+" NUKE_BUILD_ARCHS "${CMAKE_OSX_ARCHITECTURES}")
    else()
        set(NUKE_BUILD_ARCHS "${CMAKE_HOST_SYSTEM_PROCESSOR}")
    endif()
elseif(WIN32)
    set(NUKE_BUILD_ARCHS "x64")
else()
    set(NUKE_BUILD_ARCHS "${CMAKE_HOST_SYSTEM_PROCESSOR}")
endif()
add_compile_definitions(NUKE_BUILD_ARCHS="${NUKE_BUILD_ARCHS}")
message(STATUS "================================================================")
message(STATUS " NukeEngine build architectures: ${NUKE_BUILD_ARCHS}")
message(STATUS " Run dir: NukeEngine/${NUKE_RUN_SUBDIR}/<Config>")
message(STATUS "================================================================")

# MSVC defines _DEBUG in debug builds and the tree keys per-config behavior on it (the
# editor's build-config detection, the renderer's validation opt-in); clang/gcc don't — align.
if(NOT MSVC)
    add_compile_definitions($<$<CONFIG:Debug>:_DEBUG>)
endif()

# CMake-generated .vcxproj files carry no <UseDebugLibraries> by default, and the user-wide
# vcpkg MSBuild integration reads exactly that flag to pick installed/.../lib vs debug/lib —
# without it a Debug module SILENTLY links RELEASE vcpkg libs (mixed CRTs; boost::filesystem
# objects crossing that seam corrupt and crash). Emit the flag per config for every target.
set(CMAKE_VS_USE_DEBUG_LIBRARIES "$<CONFIG:Debug>")

# ---- NukeGen: the native reflection generator, FIRST in the build -----------------------
# Every reflection .gen.inc/.gen.cpp in the tree is produced by this exe (nukegen.py stays
# only for old game projects' scaffolded CMakeLists). It deploys itself to NukeUtils/bin/,
# where the engine prebuild, every module and the game-module scaffold expect it.
if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/NukeUtils/NukeGen/CMakeLists.txt")
    add_subdirectory(NukeUtils/NukeGen)
    message(STATUS "NukeGen: included (native reflection generator)")
    # Fresh checkout, single-config generators: the module .gen.inc custom commands DEPEND on
    # the DEPLOYED NukeUtils/bin/NukeGen path, which only appears through NukeGen's POST_BUILD
    # copy — ninja refuses the graph ("no known rule to make it") before ever building NukeGen.
    # Bootstrap the deployed copy once at configure time; incremental rebuilds then ride the
    # normal target + POST_BUILD flow.
    if(NOT WIN32 AND NOT EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/NukeUtils/bin/NukeGen${NUKE_HOSTTOOL_SUFFIX}")
        message(STATUS "NukeGen: bootstrapping NukeUtils/bin/NukeGen (fresh checkout)")
        file(MAKE_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/NukeUtils/bin")
        execute_process(
            COMMAND "${CMAKE_CXX_COMPILER}" -std=c++20 -O2
                    "${CMAKE_CURRENT_SOURCE_DIR}/NukeUtils/NukeGen/nukegen.cpp"
                    -o "${CMAKE_CURRENT_SOURCE_DIR}/NukeUtils/bin/NukeGen${NUKE_HOSTTOOL_SUFFIX}"
            RESULT_VARIABLE _nuke_gen_rc ERROR_VARIABLE _nuke_gen_err)
        if(NOT _nuke_gen_rc EQUAL 0)
            message(FATAL_ERROR "NukeGen bootstrap failed:\n${_nuke_gen_err}")
        endif()
    endif()
endif()

# ---- engine + editor ---------------------------------------------------------------------
# Windows: the hand-maintained solution, built in place (msbuild). Non-Windows (Deuterium):
# the engine core builds through its own CMakeLists as a shared library; the editor/player
# hosts join as they are ported. NukeEngineSln stays the ordering anchor on both paths.
if(WIN32)
    add_custom_target(NukeEngineSln ALL
        COMMAND "${CMAKE_VS_MSBUILD_COMMAND}" "${CMAKE_CURRENT_SOURCE_DIR}/NukeEngine/NukeEngine.sln"
                /t:Build /p:Configuration=$<CONFIG> /p:Platform=x64 /m /nr:false /v:m /nologo
        COMMENT "NukeEngine.sln (engine + editor) — $<CONFIG>"
        VERBATIM)
else()
    add_subdirectory(NukeEngine)
    add_custom_target(NukeEngineSln ALL)          # ordering anchor: modules depend on it
    add_dependencies(NukeEngineSln NukeEngine)
    # The editor host joins through its own CMakeLists (on Windows the sln builds it).
    if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/NukeEngine-Editor/CMakeLists.txt"
       AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/NukeImGui/CMakeLists.txt")
        add_subdirectory(NukeEngine-Editor)
        message(STATUS "NukeEngine-Editor: included (CMake host build)")
    endif()
endif()

# ---- optional modules/hosts: each is its own CMake project, included when present --------
set(NUKE_MODULES
    NukeImGui            # editor UI dll (editor links it)
    NukeRenderDiligent   # the renderer
    NukeScript           # Lua scripting
    NukeGUI              # runtime GUI
    NukePhysicsJolt      # physics
    NukeAudio            # audio
    NukeCSharp           # C# scripting
    NukeGamepad          # gamepad input provider (GLFW) — separate module by design
    NukeVFX              # particle VFX (7.3) — separate module by design
    NukeWater            # water (7.5) — separate module by design (UE-style water plugin)
    NukeTilemap          # grid worlds (colony sims / roguelikes) — separate module by design
    NukeTilemapEditor    # editor tooling companion (.nutile editor) — editor-only, never ships
    NukeAtlasImporter    # .atlas (libgdx/TexturePacker) importer — editor-only, never ships
    TestNUKEModule       # the sample plugin
    NukePlayer           # the runtime host
)
# Module CMakeLists are being ported platform by platform: NUKE_MODULES_PORTED names the
# ones whose build already works off-Windows; the rest are skipped there with a notice.
set(NUKE_MODULES_PORTED
    NukeImGui
    NukeRenderDiligent
    NukeScript
    NukeGUI
    NukePhysicsJolt
    NukeAudio
    NukeCSharp
    NukeGamepad
    NukeVFX
    NukeWater
    NukeTilemap
    NukeTilemapEditor
    NukeAtlasImporter
    TestNUKEModule
    NukePlayer
)   # (Deuterium: modules join here as their CMakeLists are ported)
foreach(m ${NUKE_MODULES})
    if(NOT EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/${m}/CMakeLists.txt")
        message(STATUS "Nuke module: ${m} — not present, skipped")
    elseif(WIN32 OR m IN_LIST NUKE_MODULES_PORTED)
        add_subdirectory(${m})
        message(STATUS "Nuke module: ${m} — included")
    else()
        message(STATUS "Nuke module: ${m} — not ported to this platform yet, skipped")
    endif()
endforeach()

# ---- build order ---------------------------------------------------------------------
# NukeImGui is engine-independent and the EDITOR links its import lib -> it builds BEFORE
# the sln; every other module links NukeEngine.lib -> they build AFTER the sln.
if(TARGET NukeImGui)
    add_dependencies(NukeEngineSln NukeImGui)
endif()
# NukeGen precedes the sln (the engine's own prebuild regenerates Reflect.gen.cpp with it);
# the modules inherit the ordering through their NukeEngineSln dependency.
if(TARGET NukeGen)
    add_dependencies(NukeEngineSln NukeGen)
endif()
foreach(m ${NUKE_MODULES})
    if(TARGET ${m} AND NOT m STREQUAL "NukeImGui")
        add_dependencies(${m} NukeEngineSln)
    endif()
endforeach()
# Every plugin exports the same C symbol `plugin` (the loader contract), and companion
# modules link each other directly (water -> renderer, tilemap editor -> tilemap). ELF's
# flat per-scope lookup then has a trap: when a companion loads FIRST, its dependency's
# internal `plugin` references resolve against the COMPANION's export — the dependency's
# constructor builds the wrong object and its own stays zeroed (crash at the first virtual
# call), depending on directory readdir order. Windows imports never interpose and Mach-O
# is two-level, so only ELF needs the symbolic pin.
if(UNIX AND NOT APPLE)
    foreach(m ${NUKE_MODULES})
        # NukePlayer is an exe. NukeImGui is NOT a dlopen'd plugin — the editor exe links it
        # directly and shares imgui's GImGui context pointer through an ELF COPY RELOCATION;
        # -Bsymbolic would pin the library to its own GImGui while the exe reads the copy
        # (CreateContext lands in one, GetActiveID reads the other — null-deref on the first
        # project frame). The plugin-collision hazard it exists for doesn't apply here anyway.
        if(TARGET ${m} AND NOT m STREQUAL "NukePlayer" AND NOT m STREQUAL "NukeImGui")
            target_link_options(${m} PRIVATE "-Wl,-Bsymbolic")
        endif()
    endforeach()
endif()

# Editor tooling companions link their runtime module's import lib — build after it.
if(TARGET NukeTilemapEditor AND TARGET NukeTilemap)
    add_dependencies(NukeTilemapEditor NukeTilemap)
endif()
# The water module drives the Diligent renderer through its native hatch (same companion
# pattern): it links the renderer's import lib — build after it.
if(TARGET NukeWater AND TARGET NukeRenderDiligent)
    add_dependencies(NukeWater NukeRenderDiligent)
endif()

# ---- self-contained editor bundle (macOS) --------------------------------------------
# Last step of every build: mirror the runtime into NukeEngine-Editor.app so the .app works
# copied to /Applications alone. Dev unaffected: RunRoot()/rpath prefer the loose run dir.
if(APPLE AND TARGET NukeEngine-Editor)
    add_custom_target(NukeEditorBundle ALL
        COMMAND /bin/sh "${CMAKE_CURRENT_SOURCE_DIR}/NukeUtils/bundle_editor.sh"
                "${CMAKE_CURRENT_SOURCE_DIR}/NukeEngine/${NUKE_RUN_SUBDIR}/$<CONFIG>"
        COMMENT "Editor bundle: syncing runtime into NukeEngine-Editor.app (self-contained)"
        VERBATIM)
    add_dependencies(NukeEditorBundle NukeEngine-Editor NukeEngineSln)
    foreach(m ${NUKE_MODULES})
        if(TARGET ${m})
            add_dependencies(NukeEditorBundle ${m})
        endif()
    endforeach()
endif()

# ---- self-contained editor AppDir (Linux) --------------------------------------------
# The same last step for Linux: mirror the runtime into NukeEngine-Editor.AppDir, flat and
# appimagetool-ready. `bundle_editor_linux.sh <rundir> --appimage <out>` squashes the image
# for distribution; the per-build target only refreshes the AppDir (cheap rsync).
if(UNIX AND NOT APPLE AND TARGET NukeEngine-Editor)
    add_custom_target(NukeEditorAppDir ALL
        COMMAND /bin/sh "${CMAKE_CURRENT_SOURCE_DIR}/NukeUtils/bundle_editor_linux.sh"
                "${CMAKE_CURRENT_SOURCE_DIR}/NukeEngine/${NUKE_RUN_SUBDIR}/$<CONFIG>"
        COMMENT "Editor AppDir: syncing runtime into NukeEngine-Editor.AppDir (self-contained)"
        VERBATIM)
    add_dependencies(NukeEditorAppDir NukeEngine-Editor NukeEngineSln)
    foreach(m ${NUKE_MODULES})
        if(TARGET ${m})
            add_dependencies(NukeEditorAppDir ${m})
        endif()
    endforeach()
endif()
