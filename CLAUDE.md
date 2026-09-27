# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A static recompilation of Rock Band 3 (Xbox 360, title ID `45410914`) built on the [ReXGlue SDK](https://github.com/rexglue/rexglue-sdk). The `rexglue` codegen translates the game's PowerPC `default.xex` into C++ (`generated/`), and the code in `src/` overrides and patches individual guest functions. The function map was built against the Title Update 5 xex, but the disc xex also boots with it.

## SDK version

The project is pinned to **rexglue 0.8.0**: `sdk_version` in `band3_manifest.toml`, and `find_package(rexglue 0.8.0)` in the generated `generated/rexglue.cmake`. The SDK is installed outside the repo and found through `CMAKE_PREFIX_PATH`. This machine has two installs:
- `C:\exes\rexglue`: the prebuilt release. It has D3D12 only (Vulkan off).
- `C:\exes\rexglue-vk`: built from source (`C:\exes\rexglue-src`, tag v0.8.0) with `REXGLUE_USE_VULKAN=ON` and `REXGLUE_USE_D3D12=ON`. **The build is configured against this one.**

To rebuild the SDK from source on Windows, check for git symlinks first. With `core.symlinks=false`, they check out as text stubs holding the target path, which fails the build (for example `lzxd.c:1:1: error: expected identifier`). Find them with `git ls-files -s | grep ^120000` in the repo and in each submodule, and replace each stub with a copy of its target.

If `generated/` was produced by a different SDK version, CMake fails with "ReXGlue SDK not found". Regenerate it with the matching `rexglue.exe`.

rexglue 0.10 breaks things silently. The GPU becomes a runtime plugin (`rexgpu-xenos.dll`) that is off by default, which gives a black window and the log line `no GPU emulation loaded (gpu_plugin not set)`. Upgrading needs `rexglue_setup_target(band3 GPU_PLUGINS xenos)` plus setting `config.gpu_plugin = "xenos"` in `OnPreSetup`.

## Build

Use a shell with the MSVC environment loaded ("x64 Native Tools Command Prompt", or `vcvars64.bat` from VS 18 BuildTools). The compiler is clang++ from VS's LLVM, and Ninja is the generator.

```bat
:: needs assets\default.xex in the repo root (gitignored); takes ~2 min
C:\exes\rexglue-vk\bin\rexglue.exe codegen band3_manifest.toml

cmake --preset win-amd64-release -DCMAKE_PREFIX_PATH=C:/exes/rexglue-vk
cmake --build --preset win-amd64-release -j 4
```

- Always limit parallelism (`-j 4` on a 16 GB machine). The ~100 generated `band3_recomp.N.cpp` files are 2+ MB each, and an unlimited Ninja build has run the machine out of memory and crashed it.
- The build also re-runs codegen when its inputs change (target `band3_codegen`). Editing `band3_config.toml` or `band3_manifest.toml` therefore triggers a full rebuild. Editing only `src/` relinks in seconds.
- Presets: `win-amd64-{debug,release,relwithdebinfo}` and `linux-amd64-*`. Output goes to `out/build/<preset>/`.
- There are no tests or linters.
- `flake.nix` is stale: it runs `codegen band3_config.toml`, which predates the manifest.

## Running

Run from `out/build/<preset>/`. Paths resolve relative to the working directory:
- `assets/`: the game dump (`default.xex`, `gen/*.ark`). This is `game_data_root`, and it has to be copied into the build folder.
- `band3_config.ini`: the project's settings. The copy in the repo root is only the template. The copy **next to the exe is the one that's read**, so apply runtime settings there, and mirror new keys into the template.
- `band3.toml` (optional, gitignored): rexglue's own cvar file.
- Logs: `logs/band3_NNN.log`. Each launch writes a new file.

Errors that are normal in the log:
- `update:\gen` / `patch_xbox.hdr` not found: no title-update folder is mounted, so the game falls back to `gen/main_xbox.hdr`.
- `Xam*`/`XAudio*` STUB warnings.
- `game:\Content\0000000000000000` not found.
- missing `gamecontrollerdb.txt`.

**GPU backend.** The SDK's `SetupPresentation` picks D3D12 whenever it's compiled in. `OnPreSetup` overrides that with `config.graphics = REX_GRAPHICS_BACKEND(VulkanGraphicsSystem)` when the ini has `[graphics] backend = vulkan`. D3D12 is the backend that works. Vulkan renders with a red tint and logs invalid fetch constant warnings. The `[rexglue] vulkan_device` index is in the app's own order, which is the reverse of `vulkaninfo` on this machine: 0 is the RTX 4060 and 1 is the Intel iGPU.

Working settings (D3D12): `readback_resolve = fast`, `render_target_path_d3d12 = rtv`, `disable_approximate_lights = false`.

## Architecture

**Function map → hookable symbols.**
- `band3_config.toml` lists about 67k guest functions as `0xADDR = { name, size }`.
- It also holds `[[midasm_hook]]` entries (mid-function register patches), `[[switch_tables]]`, and a `[rexcrt]` section of CRT replacements, mostly commented out.
- Named functions become C++ symbols. To override one, define `extern "C" REX_FUNC(Name)` in `src/`. To call the original from inside the override, declare `extern "C" void __imp__Name(PPCContext&, uint8_t*)` and call it.
- Guest memory is accessed through `base` with `REX_LOAD_U32`/`REX_STORE_U32` and similar macros. Arguments and return values go through `ctx.r3`, `ctx.r4`, etc.
- Midasm hooks are plain functions in `src/patches.cpp` that take `PPCRegister&` for the registers named in the toml.
- To see what a function does, read `generated/band3_recomp.N.cpp`, where each PPC instruction appears as a comment above its C++. `generated/band3_init.cpp` holds the address→name table.

**Code layout:**
- `src/band3_app.h`: `Band3App : rex::ReXApp`. Lifecycle hooks, in order: `OnConfigurePaths` (loads the ini), `OnPreSetup` (runs before the GPU is created, and applies the ini's `[rexglue]` section as rexglue cvars via `rex::cvar::SetFlagByName`, logging each), `OnPostSetup`, `OnCreateDialogs` (debug FPS overlay).
- `src/config.{h,cpp}`: reads `band3_config.ini` with inih into `band3::Config`. `LoadConfig` also injects game command-line args (`-lang`, `-fast`, `-define MHX_PC`). The `OptionBool` and `OptionStr` hooks in `patches.cpp` serve those args to the game. `LoadConfig` runs again on every `MetaPerformer::SetVenue`, so venue settings can change without restarting. To add a setting, add a field to `Config`, a read in `config.cpp`, and the key to both ini copies.
- `src/patches.cpp`: gameplay and boot patches (debugger trap, disk-error and checksum bypasses, metamusic toggle, demo flag, forced venue, controller type, song count).
- `src/Hooks/`: overrides grouped by subsystem:
  - `graphics.cpp`: lighting, material and even/odd-frame workarounds, controlled by the ini's `[graphics]` section
  - `file.cpp`: ARK paths
  - `memory.cpp`: heap sizes
  - `crypto.cpp`: XeKeys
  - `profile.cpp`: username
  - `camera_shake.cpp`: a hand-translated, frame-rate independent `CamShot::Shake`. Turn it off with `[debug] native_camera_shake`.
  - `math.cpp`: native vector, matrix and libm replacements. Turn them off with `[debug] native_math`. Each hook falls back to `__imp__`, except `_sin`, which isn't in the function map and so has no `__imp__`.

**Hand-copied guest code is tied to one xex version.** `camera_shake.cpp` was copied from the TU5 xex. The disc xex has identical code, but its constant data sits at different addresses. The hard-coded `lfs`/`lfd` displacements made the camera matrix garbage, which turned the whole 3D scene (characters and venues) black while the UI still drew. The fix reads each displacement from the game's own instruction at runtime, and falls back to the original if the opcode doesn't match. Any code that copies guest instructions must not hard-code data offsets or `lis`/`addi` addresses. When 3D rendering breaks, bisect with the `native_*` toggles before blaming the GPU.
- `src/Game/`: host-side mirrors of the game's scripting engine types (`DataNode`, `DataArray`, `Symbol`, `BinStream`) that read and write them in guest memory. `DTAFunctions.cpp` adds custom DTA script functions (such as `exit`) by hooking `DataArray::Execute` and dispatching on the first symbol.

Known quirk: `config.cpp` reads `game_data_root` from `[paths]`, but the ini has it under `[game]`. It only works because both default to `assets`.

Known bug: `band3_config.toml` `[rexcrt]` maps both `strchr` and `strrchr` to `0x8282C3A0`. Codegen binds `rexcrt_strrchr` there, so the roughly 9 `strchr` call sites (all path checks) get `strrchr`.
