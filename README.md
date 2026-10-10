# Hyprland Cloth Cursor

Native Hyprland C++ plugin to give your cursor inertial drag, rotation, bend, stretch, and click squash!

https://github.com/user-attachments/assets/21f9e71b-1e15-45f2-aabc-06c6c4872f2d

The compiled version of the plugin is "libclothcursor.so". The installer appends a small startup entry to lua for end4 machines which load and enable the plugin on login.

> [!IMPORTANT]
> This release is live-tested against these exact Hyprland builds:
>
> ```text
> 0.55.4  a0136d8c04687bb36eb8a28eb9d1ff92aea99704_aq_0.12_hu_0.13_hg_0.5_hc_0.1_hlg_0.6
> 0.56.1  5c9377c15f85c50648f35ca5a213754f95b93ca0_aq_0.14_hu_0.14_hg_0.5_hc_0.1_hlg_0.6
> 0.56.2  efb50993780079460b0cbed1363e2166a2de1d9f_aq_0.14_hu_0.14_hg_0.5_hc_0.1_hlg_0.6
> 0.56.2  efb50993780079460b0cbed1363e2166a2de1d9f_aq_0.15_hu_0.14_hg_0.5_hc_0.1_hlg_0.6
> ```
>
> The others are **untested**, but can be compiled with the installed headers with an explicit warning and confirmation. It still does not allow a build/runtime ABI mismatch, and will restore/use the original cursor if the protected renderer-hook checks do not pass. (Currently using OpenGL as the renderer.)

## What it does

 - Moves visible cursor around bounded 2D spring.
 - Calculate rotation / skew / stretch from spring lag & velocity.
 - When a mouse button is held it squashes cursor down to `79%` and springs back.
 - Transforms the cursor texture given by Hyprland (including Theme, App cursors)
 - Preserve app input at Hyprland's real pointer location.
 - Only update previous and current transformed cursor regions.
 - Returns to Hyprland stock cursor when disabled or rendering can't be done.
  
The cursor visible follows the pointer in fast motion. Clicks still use the actual pointer position.

## How it works

<img width="1448" height="1086" alt="b81d1ca1-71b3-457d-9041-46d140093651" src="https://github.com/user-attachments/assets/97a37453-1910-4459-9811-d8ca209598cd" />


### 1. Pointer events set the target

Spring target is updated by mouse movement. The event handler then damages the affected area of the cursor and schedules a compositor frame.

### 2. A spring moves the visible cursor

The spring stores:

```text
visual position
visual velocity
```

Its acceleration is calculated from displacement and damping:

```text
acceleration =
    stiffness × (target - visual position)
    - damping × visual velocity
```

Simulation is done in tiny `(1/240s)` integration steps with bounded frame time, max lag and max velocity. It snaps to the target once the settling thresholds are met.

### 3. Motion creates the cloth effect

The plugin gets the visual transform from spring state:
- horizontal lag,velocity gives rotation
- same motion signal gives smaller skew
- lag distance, speed gives stretch
- cross-axis compression to balance shape

All values have fixed limits.

### 4. A second spring handles clicks

Mouse button state controls a scalar spring from 0 to 1. Its will scale the values of both cursor axes down during a press and restores them on release.

### 5. Hyprland draws the transformed cursor

The plugin hooks:

```text
CPointerManager::renderSoftwareCursorsFor(...)
```

Basically it just reads hyprland's active cursor texture, size, scale and hotspot. And it submits a custom render-pass element. It applies the transform of the cursor around the cursor hotspot, in this order:

```text
rotation => skew => scale
```


The plugin forces software cursor rendering when enabled. This is because the transformed texture is drawn inside the compositor render pass.

### 6. Damage follows the transformed shape

The plugin computes a conservative bounding box for all 4 cursor corners after the transformation. It damages the union of the old and new bounding boxes (plus a small padding around it). This clears the old cursor image and prevents full-screen redraws.

## Project structure

| Path | Role |
|---|---|
| `src/plugin.cpp` | Plugin entrypoint and runtime ABI verification |
| `src/runtime.cpp` | Hyprland hooks, input listeners, frame scheduling, controls, fallback handling |
| `src/hyprland_compat.hpp` | Compatibility bridge for Hyprland 0.55 and 0.56 pointer/monitor APIs |
| `src/physics.cpp` | Spring simulation, visual transform, hotspot anchoring, transformed bounds |
| `src/cursor_pass.cpp` | Hyprland render-pass element and texture transform |
| `tests/physics_test.cpp` | Physics and geometry tests |
| `tests/nested_multi_output_test.sh` | Isolated bidirectional two-output render/scheduling gate |
| `scripts/clothcursorctl` | Load, enable, disable, status, and unload commands |
| `install.sh` | Local build, tests, installation, startup entry, rollback |
| `cmake/VerifyHyprland.cmake` | Build-time Hyprland ABI check |

## Installation

The installer compiles the plugin locally on your machine, tests it, and installs it in ~/.local/, and also creates a managed startup.

```bash
git clone https://github.com/bg-l2norm/hyprland-cloth-cursor.git
cd hyprland-cloth-cursor
./install.sh
```

Required tools:

- CMake 3.27 or newer
- C++23 compiler
- Hyprland development headers
- `pkg-config`
- `python3`
- `hyprctl`
- `flock`
- `readelf`
- `ldd`

On Arch Linux, these come from packages such as `base-devel`, `cmake`, `hyprland`, `util-linux`, `binutils`, and `glibc`.

The installer:

1. checks the installed Hyprland ABI and warns before continuing on an untested build;
2. builds `libclothcursor.so`;
3. runs the physics tests;
4. checks the shared library and its dependencies;
5. installs the plugin and controller under `~/.local/`;
6. adds one managed startup block to `custom/execs.lua` or `custom/execs.conf`;
7. loads and enables the plugin in the current session.

Repeated installation updates the same files and startup block. Edited configuration files receive timestamped backups.

## Runtime controls

```bash
clothcursorctl status
clothcursorctl enable
clothcursorctl disable
clothcursorctl toggle
clothcursorctl unload
```

- `disable` restores the stock cursor and keeps the shared library loaded.
- `unload` disables the effect and removes the plugin from the current Hyprland session.
- `status` reports installation, load, and runtime state.

The plugin also registers:

```bash
hyprctl clothcursor status
hyprctl clothcursor enable
hyprctl clothcursor disable
hyprctl clothcursor toggle
```

## Startup on end4

For a Lua-based end4 configuration, the installer adds this managed behavior to `custom/execs.lua`:

```lua
hl.on("hyprland.start", function()
    hl.exec_cmd("~/.local/bin/clothcursorctl session-start")
end)
```

The startup entry launches the controller. The cursor simulation and rendering run inside the compiled C++ plugin.

Legacy Hyprland `.conf` configuration uses an equivalent `exec-once` entry.

## Cursor themes

Hyprland provides the active cursor texture. The cloth cursor modifies that texture during the rendering process. 

This allows it to remain compatible with changing cursor themes and sizes. Cursor change events trigger damage for the old and new areas so the new shape is shown right away.

## Disable and uninstall

Restore the stock cursor while keeping the plugin loaded:

```bash
clothcursorctl disable
```

Unload it from the current session:

```bash
clothcursorctl unload
```

Remove the installed files and managed startup entry:

```bash
./install.sh uninstall
```

The uninstaller disables and unloads the plugin before deleting its installed library and controller.

## Safety and limitations

- disabled by default after direct loading. enable via `clothcursorctl enable`
- tested baseline is hyprland versions 0.55.4, 0.56.1 and 0.56.2 at exact commits as above. feel free to try other builds after verifying, but it's not guaranteed to work.
- renderer path: needs hyprland's opengl renderer
- HDR/ICC + other renderer paths are not in this release
- software cursor compositing uses more power than hw cursor plane
- cursor lags behind input if moving fast
- disconnecting the mouse while holding down a button could keep the press compression enabled. to reset:
```bash
clothcursorctl disable
clothcursorctl enable
```
- controller based disable/unload will remove pending custom cursor passes and release the software cursor lock
- a plugin fault runs inside the compositor process, it could kill the hyprland session

## Development build

```bash
cmake -S . -B build -DCMAKE_BUILD_TYPE=RelWithDebInfo
cmake --build build -j2
ctest --test-dir build --output-on-failure

hyprctl plugin load "$PWD/build/libclothcursor.so"
hyprctl clothcursor enable
```

Build only the physics library and tests:

```bash
cmake -S . -B build-physics \
    -DCLOTH_CURSOR_BUILD_PLUGIN=OFF \
    -DCMAKE_BUILD_TYPE=Debug

cmake --build build-physics
ctest --test-dir build-physics --output-on-failure
```

Enable AddressSanitizer and UndefinedBehaviorSanitizer:

```bash
cmake -S . -B build-sanitize \
    -DCLOTH_CURSOR_SANITIZE=ON \
    -DCMAKE_BUILD_TYPE=Debug

cmake --build build-sanitize
ctest --test-dir build-sanitize --output-on-failure
```

With a parent Hyprland session up and running, run the isolated two-output compositor gate:
```bash
./tests/nested_multi_output_test.sh
```
This will launch a temporary nested Hyprland and add a side-by-side headless output, check A->B + B->A spring settle and owner output scheduling, insist on no render rejects/fallbacks, and finally unload plugin and exit nested compositor.

## License

MIT — see [`LICENSE`](LICENSE).
