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
project(NukeAll NONE)

if(NOT CMAKE_GENERATOR MATCHES "Visual Studio")
    message(FATAL_ERROR "The superbuild drives .vcxproj projects — use a Visual Studio generator (-G \"Visual Studio 17 2022\" -A x64)")
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

# CMake-generated .vcxproj files carry no <UseDebugLibraries> by default, and the user-wide
# vcpkg MSBuild integration reads exactly that flag to pick installed/.../lib vs debug/lib —
# without it a Debug module SILENTLY links RELEASE vcpkg libs (mixed CRTs; boost::filesystem
# objects crossing that seam corrupt and crash). Emit the flag per config for every target.
set(CMAKE_VS_USE_DEBUG_LIBRARIES "$<CONFIG:Debug>")

# ---- engine + editor: the hand-maintained solution, built in place -----------------------
add_custom_target(NukeEngineSln ALL
    COMMAND "${CMAKE_VS_MSBUILD_COMMAND}" "${CMAKE_CURRENT_SOURCE_DIR}/NukeEngine/NukeEngine.sln"
            /t:Build /p:Configuration=$<CONFIG> /p:Platform=x64 /m /v:m /nologo
    COMMENT "NukeEngine.sln (engine + editor) — $<CONFIG>"
    VERBATIM)

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
    NukeTilemap          # grid worlds (colony sims / roguelikes) — separate module by design
    NukeTilemapEditor    # editor tooling companion (.nutile editor) — editor-only, never ships
    NukeAtlasImporter    # .atlas (libgdx/TexturePacker) importer — editor-only, never ships
    TestNUKEModule       # the sample plugin
    NukePlayer           # the runtime host
)
foreach(m ${NUKE_MODULES})
    if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/${m}/CMakeLists.txt")
        add_subdirectory(${m})
        message(STATUS "Nuke module: ${m} — included")
    else()
        message(STATUS "Nuke module: ${m} — not present, skipped")
    endif()
endforeach()

# ---- build order ---------------------------------------------------------------------
# NukeImGui is engine-independent and the EDITOR links its import lib -> it builds BEFORE
# the sln; every other module links NukeEngine.lib -> they build AFTER the sln.
if(TARGET NukeImGui)
    add_dependencies(NukeEngineSln NukeImGui)
endif()
foreach(m ${NUKE_MODULES})
    if(TARGET ${m} AND NOT m STREQUAL "NukeImGui")
        add_dependencies(${m} NukeEngineSln)
    endif()
endforeach()
# Editor tooling companions link their runtime module's import lib — build after it.
if(TARGET NukeTilemapEditor AND TARGET NukeTilemap)
    add_dependencies(NukeTilemapEditor NukeTilemap)
endif()
