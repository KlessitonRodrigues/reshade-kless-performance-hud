# Performance HUD

A compact performance HUD add-on for [ReShade](https://reshade.me) that draws
CPU, GPU, memory and frame-rate statistics as a single column on ReShade's
always-visible on-screen display (OSD).

## Features

- CPU usage and optional per-logical-core usage
- GPU usage, power draw and temperature (NVIDIA via NVML)
- VRAM usage and system RAM usage
- Battery charge level
- FPS with 1% and 0.1% lows over a rolling 10 second window
- Optional "Smooth Motion" mode that doubles the FPS statistics for frame
  generation / interpolated presents
- Optional load-based coloring and a "show performance issues only" mode that
  hides healthy metrics and surfaces warnings automatically
- Configurable text scale, vertical offset, base color and refresh interval

The HUD is rendered by the add-on itself, so it does not require a ReShade
preset or effect.

## Installation

1. Install ReShade with add-on support.
2. Copy `PerformanceHUD.addon64` into the game directory, next to the ReShade
   DLL (or into a `reshade-addons` subfolder).
3. Start the game. The HUD appears in the top-left corner of the OSD.

Open ReShade's overlay and select the **Performance HUD** tab to configure the
add-on. Use **Settings > Overlay > OSD position on screen** to move the HUD to a
different corner; `HudPosY` adjusts the vertical offset in pixels.

## Building

The primary build is `./build.sh`, which cross-compiles with `llvm-mingw` and
writes `PerformanceHUD.addon64` to the project root:

```sh
./build.sh
```

The script uses `MINGW_PREFIX` if set, otherwise an
`x86_64-w64-mingw32-clang++` on `PATH`, and otherwise downloads a pinned
llvm-mingw release into `~/.cache` on first use (network required).

CMake is also supported:

```sh
cmake -B build
cmake --build build
```

Warnings about `unknown attribute 'novtable'` from the vendored ReShade headers
are expected under clang.

## Data sources and limitations

| Metric | Source |
| --- | --- |
| CPU usage | `GetSystemTimes` deltas |
| Per-core usage | `NtQuerySystemInformation` |
| RAM usage | `GlobalMemoryStatusEx` |
| Battery | `GetSystemPowerStatus` |
| GPU usage, power, temperature, VRAM | NVIDIA NVML |
| GPU usage, VRAM fallback | PDH `GPU Engine` / `GPU Adapter Memory` counters |

GPU power and temperature require an NVIDIA GPU and driver with NVML. On other
vendors the add-on falls back to Windows PDH counters, which provide GPU usage
and VRAM usage only. The PDH VRAM fallback reports the amount in use, not the
total, so it is always shown in the base color.

## Configuration

Settings are stored in `ReShade.ini` under the `[PERFORMANCE_HUD]` section and
are written when changed in the add-on settings tab.

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
| `HudScale` | float | Text scale (0.25-4.0) |
| `HudPosY` | float | Vertical HUD offset in pixels |
| `HudColor` | string | `r,g,b,a` base text color |
| `ColorLinesByLoad` | bool | Threshold colors and "show issues only" |
| `RefreshIntervalMs` | int | Sampler interval (100-10000 ms) |

## License

BSD-3-Clause OR MIT.
