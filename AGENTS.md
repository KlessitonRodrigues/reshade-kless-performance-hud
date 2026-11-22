# AGENTS.md

Guidance for coding agents working on the Performance HUD ReShade add-on.

## Project overview

A single-file C++20 ReShade add-on that renders a compact performance HUD on
ReShade's always-visible OSD. It reports CPU/GPU usage, per-core usage, GPU
power/temperature, VRAM/RAM usage, battery level, and FPS with 1% / 0.1% lows
over a rolling 10 second window.

## Layout

```
CMakeLists.txt              CMake build (optional; outputs to out/)
build.sh                    Primary build: cross-compiles with llvm-mingw
include/                    Vendored ReShade + Dear ImGui headers (do not edit)
src/performance_hud.cpp     Entire add-on implementation
out/PerformanceHUD.addon64  Build artifact (created in out/)
```

## Build

The primary build is `./build.sh`. It uses `llvm-mingw`:

1. `MINGW_PREFIX` environment variable, if set.
2. `x86_64-w64-mingw32-clang++` on `PATH`.
3. Otherwise it downloads a pinned llvm-mingw release into `~/.cache` on first
   use (requires network access).

```sh
./build.sh
```

The result is `out/PerformanceHUD.addon64`. The build is a Windows x86-64 DLL;
it cannot be executed on Linux. Success is measured by the compiler exiting
zero.

CMake is also supported (`cmake -B build && cmake --build build`) and writes to
`out/`, but `cmake` may not be installed; prefer `build.sh`.

Warnings from `include/reshade_*.hpp` about `unknown attribute 'novtable'` are
expected under clang and are not caused by project code.

There are no tests or linters configured. The build is the only verification.

## Architecture (`src/performance_hud.cpp`)

- `DllMain` calls `reshade::register_addon`, then `init()`.
- `init()` loads config, NVML, PDH and ntdll, then registers:
  - overlay `"OSD"` -> `draw_osd` (always-visible HUD),
  - overlay `nullptr` -> `draw_settings` (add-on settings tab),
  - event `reshade::addon_event::reshade_present` -> `on_present`.
- `on_present` records frame times and lazily starts a sampler thread.
- `sampler_thread` calls `update_snapshot()` every `g_interval_ms`
  (default 1000 ms, clamped 100-10000 ms).
- `update_snapshot` samples metrics and publishes a `StatsSnapshot` under
  `g_snapshot_lock` (SRWLOCK).
- `draw_osd` copies the snapshot and FPS stats, then draws one metric per line.

Data sources:

- CPU usage: `GetSystemTimes` deltas.
- Per-core usage: `NtQuerySystemInformation(SystemProcessorPerformanceInformation)`.
- RAM: `GlobalMemoryStatusEx`.
- Battery: `GetSystemPowerStatus`.
- GPU usage/power/temp/VRAM: NVML (`nvml.dll`, probed in the driver store if
  not on the search path).
- GPU usage/VRAM fallback: PDH counters `\GPU Engine(*)\Utilization Percentage`
  and `\GPU Adapter Memory(*)\Dedicated Usage`.
- FPS and percent lows: ring buffer of presentation deltas, 10 s rolling window.

Threading: the sampler thread and overlay/render thread communicate only through
SRWLock-protected snapshots and atomics. Do not join the sampler thread from
`DllMain` (loader-lock deadlock); signal its stop event instead.

## Conventions

- C++20, tabs for indentation, braces on their own line (Allman).
- File-scope globals use the `g_` prefix; compile-time constants use `k`.
- Keep everything in the single translation unit unless there is a strong
  reason to split it.
- The `include/` headers are vendored dependencies: never modify them.
- Settings persist through `reshade::get_config_value` /
  `reshade::set_config_value` under the `PERFORMANCE_HUD` section.
- Add a matching get/set pair for any new setting, and load it in `load_config`.
- Do not add comments unless they explain non-obvious intent; match the
  existing sparse-comment style.
- The add-on targets Windows only; keep Windows API usage behind the existing
  patterns and avoid POSIX dependencies.

## Configuration keys

| Key | Type | Meaning |
| --- | --- | --- |
| `ShowCPU` | bool | CPU usage line |
| `ShowCore` | bool | One line per logical core |
| `ShowGPU` | bool | GPU usage line |
| `ShowGPUTemp` | bool | GPU temperature line |
| `ShowPower` | bool | GPU power line |
| `ShowVram` | bool | VRAM usage line |
| `ShowRam` | bool | RAM usage line |
| `ShowBattery` | bool | Battery line |
| `ShowFps` | bool | FPS / 1% / 0.1% lines |
| `SmoothMotion` | bool | Double FPS stats for frame generation |
| `HudScale` | float | Text scale (0.25-4.0, whole steps in UI) |
| `HudPosY` | float | Vertical HUD offset in pixels |
| `HudColor` | string | `r,g,b,a` base text color |
| `ColorLinesByLoad` | bool | Threshold colors + "show issues only" |
| `RefreshIntervalMs` | int | Sampler interval (100-10000 ms) |
