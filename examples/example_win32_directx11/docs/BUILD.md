# Build & Run

Win32 + Direct3D 11 Glass Overlay (Dear ImGui). **x64-only**, **VS2022**.

## TL;DR

```bash
# from the project dir: examples/example_win32_directx11
MSBuild example_win32_directx11.vcxproj -p:Configuration=Release -p:Platform=x64
./Release/example_win32_directx11.exe
```

There is also a helper script (kills any running instance, builds, optionally launches):

```bash
bash scripts/build.sh        # build Release|x64
bash scripts/build.sh run    # build, then launch + verify the process is alive
```

## Requirements

| Need | Detail |
|------|--------|
| Visual Studio 2022 | Provides MSBuild + the C++ toolset. `build.sh` locates MSBuild automatically — first on `PATH`, then any installed VS 2022/2019 edition (Community / Professional / Enterprise / Build Tools). |
| Platform toolset `v143` | Pinned in the `.vcxproj`. |
| Windows SDK `10.0.22621.0` | Pinned in the `.vcxproj` (`WindowsTargetPlatformVersion`). |
| C++17 | `LanguageStandard = stdcpp17`. |
| x64 only | The bundled FreeType ships an **x64** import lib only (`thirdparty/freetype/win64/freetype.lib`) and `IMGUI_ENABLE_FREETYPE` is on, so a Win32/x86 build cannot link. The `.vcxproj` only declares `Debug|x64` and `Release|x64`. |

System libs are pulled in via the project / `#pragma comment`: `d3d11`, `d3dcompiler`, `dxgi`, `freetype`, plus `dwmapi`, `windowscodecs`, `ole32`, `winmm`. No package manager / external fetch step.

> The subsystem is **Console** (`/SUBSYSTEM:CONSOLE`), so launching from a terminal keeps a console window for stdout. This is intentional, not a bug.

## What actually compiles (unity build)

Despite a large `src/` tree, **only four `.cpp` files are compiled** (the four `<ClCompile>` entries under `src/` in the `.vcxproj`):

```
src/main.cpp
src/glass/glass.cpp
src/glass/surfaces.cpp
src/glass/backdrop.cpp
```

(Plus the upstream ImGui core + backends: `imgui.cpp`, `imgui_draw.cpp`, `imgui_tables.cpp`, `imgui_widgets.cpp`, `misc/freetype/imgui_freetype.cpp`, `backends/imgui_impl_dx11.cpp`, `backends/imgui_impl_win32.cpp`.)

**Everything else is `#include`d as `.inc` fragments, not compiled separately.** If you add a new `.inc`/widget file, do **not** add it to the project as a `<ClCompile>` — `#include` it from the appropriate aggregator instead. Key inclusion points inside `main.cpp`:

| Included file | Where | What it is |
|---------------|-------|------------|
| `app/megasettings.inc` | top of `main.cpp` | data-driven global settings catalog (~11,108 controls) + `DrawMegaSettings()` |
| `app/config.inc` | top of `main.cpp` | `glass.cfg` loader/writer |
| `app/pages.inc` | **inside** `main()`'s render loop | the dashboard pages/UI |
| `app/render_capture.inc` | bottom of `main.cpp` | D3D device/RTV setup, MP4 recorder, SSAA resolve — aggregates `app/render/{device_d3d11, recorder_mp4, ssaa_resolve}.inc` |

`glass.cpp` similarly `#include`s the widget impls under `src/glass/widgets/`. Order matters: fragments are pulled in dependency order, so edit the includer, not the project file.

## Build configurations

| Config | Optimization | Notes |
|--------|--------------|-------|
| `Release` | `MaxSpeed`, `WholeProgramOptimization`, COMDAT folding, intrinsics, `BufferSecurityCheck` off | what you ship |
| `Debug` | `Disabled` | symbols on; for stepping |

Both emit to `./$(Configuration)/` (`Release\` or `Debug\`), e.g. `Release/example_win32_directx11.exe`. Both set warning level 4 and `/utf-8`.

## Runtime dependencies

The exe is designed to run off **any** machine, not just the dev box. Asset/config paths are resolved at runtime; nothing is hard-required except the embedded fonts.

### Fonts

- **Inter (medium + semibold)** — **embedded in the binary** as byte arrays in `data/fonts.h` (`#include "../../../data/fonts.h"`), loaded via `AddFontFromMemoryTTF`. Always present; no file needed.
- **FontAwesome 6** (`data/fa/fa-solid-900.ttf`, `data/fa/fa-brands-400.ttf`) and **Sacramento** (`data/Sacramento-Regular.ttf`) — loaded from **disk** via `AddFontFromFileTTF`. Resolved through `AssetPath()` (see below). If a file is missing, icons degrade gracefully (procedural fallback) rather than crashing — but ship the `data/` folder for the intended look.

#### `AssetPath()` probing order

For a relative asset like `data\fa\fa-solid-900.ttf`, the lookup is:

1. **exe directory**, then **up to 6 parent folders** (the build output sits a few folders below the repo root, so the in-repo `data/` is found automatically during dev).
2. `%GLASS_ASSET_DIR%\<relative path>` — env override.
3. The relative path itself, as a last resort (resolved against the process working directory).

To run elsewhere: copy the exe next to (or above) a `data/` folder, **or** set `GLASS_ASSET_DIR` to the folder that contains `data\`.

### `glass.cfg` (configuration)

A fully-commented `glass.cfg` is **written next to the exe on first run** if none is found. It is a simple `key = value` INI: window size/pos, appearance/accent, frosted material, and edge-shader params. Every field has a default equal to the hard-coded look, unknown keys are ignored, and numeric values are range-clamped — a missing or malformed file can't crash the app.

**Resolution order:** `%GLASS_CONFIG%` → `<exe dir>\glass.cfg` → walk up to 6 parent folders. If none exists, a default is written to `<exe dir>\glass.cfg`. "Save" in-app writes back to whichever file was loaded (or a fresh one next to the exe). Per-slot presets are `glass_preset_N.cfg` next to the exe.

### ffmpeg (optional — F9 recorder only)

The in-app MP4 recorder (toggle with **F9**, auto-stops after 2 minutes) pipes raw BGRA frames to a child `ffmpeg` process. It is fully optional; if ffmpeg can't be launched, recording silently no-ops.

ffmpeg lookup: try `C:\ffmpeg\ffmpeg.exe` first; if absent, fall back to `ffmpeg.exe` on the system **PATH** (`CreateProcess` searches PATH). Output defaults to `%USERPROFILE%\glass_recording.mp4`. ffmpeg's stdout/stderr go to `glass_ffmpeg_log.txt` in the temp dir.

## Environment variable overrides

| Var | Effect |
|-----|--------|
| `GLASS_ASSET_DIR` | Base folder for on-disk assets (must contain `data\...`). Checked after exe-dir/parent probing, before the dev fallback. |
| `GLASS_CONFIG` | Full path to a specific `glass.cfg` to load (highest priority in config resolution). |

## Common issues

- **Link error about Win32/x86** — you tried to build `Win32`. Use `-p:Platform=x64`; this project is x64-only (FreeType import lib).
- **MSBuild not found** — `build.sh` hard-codes the *Community* path. For other VS editions, run `MSBuild` from a *Developer Command Prompt* or fix the path in `scripts/build.sh`.
- **Icons look like plain shapes / no cursive "hello"** — the `data/` font files weren't found. Place `data\` next to the exe or set `GLASS_ASSET_DIR`.
- **F9 does nothing** — ffmpeg isn't at `C:\ffmpeg\ffmpeg.exe` and isn't on PATH. Install ffmpeg or set one of those.
- **Adding a source file did nothing / duplicate-symbol errors** — remember the unity build: `#include` new `.inc` files from the right aggregator; only `main.cpp` / `glass.cpp` / `surfaces.cpp` / `backdrop.cpp` are compiled.
