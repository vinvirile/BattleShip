# `osContInit` filesystem_error from CI-baked install prefix (2026-05-03)

## Symptom

Windows users running the v0.7.2 release crashed during boot at `osContInit`,
right after `Audio initialized` and before the controller subsystem could
register. The user log captured the C++ throw the port's vectored handler
unwound:

```
SSB64: osCreateThread id=6 entry=…  (registered=6 total)

*** C++ THROW (first-chance) tid=… ***
    C++ throw: type="class std::filesystem::filesystem_error" thrown=…
        what=exists: The device is not ready.:
             "D:/a/BattleShip/BattleShip/BattleShip/gamecontrollerdb.txt"

*** C++ THROW (first-chance) tid=… ***
    C++ throw: type="class spdlog::spdlog_ex" thrown=…
        what=async flush: thread pool doesn't exist anymore
```

Battleship.log:

```
[critical] Exception: 0xe06d7363
```

`0xE06D7363` is the MSVC C++ throw exception code, raised here as a
first-chance exception that nobody catches. The `async flush` follow-up is
spdlog's async pool getting torn down on the way out as the runtime begins
process termination.

The reported path — `D:/a/BattleShip/BattleShip/BattleShip/gamecontrollerdb.txt`
— is the **GitHub Actions runner's workspace**. No user has that drive
layout; on most Windows machines `D:` is an empty optical/SD reader.

## Root cause

Two compounding bugs ship in the Windows release zip:

**1. CMAKE_INSTALL_PREFIX gets baked in absolute by CI.**
`scripts/package-windows.ps1` runs `cmake -DCMAKE_INSTALL_PREFIX=BattleShip`
from the runner's checkout root (`D:\a\BattleShip\BattleShip`). CMake
resolves a relative `CMAKE_INSTALL_PREFIX` against the current working
directory at configure time, so libultraship's `install_config.h` ends up
with the **absolute** runner path:

```
#define CMAKE_INSTALL_PREFIX "D:/a/BattleShip/BattleShip/BattleShip"
#define NON_PORTABLE
```

`Ship::Context::GetAppBundlePath()` under `NON_PORTABLE` returns that
literal at runtime — meaningless on the user's machine.

**2. `Ship::Context::LocateFileAcrossAppDirs` uses the throwing
`std::filesystem::exists` overload.**

```cpp
// libultraship/src/ship/Context.cpp (pre-fix)
std::string Context::LocateFileAcrossAppDirs(...) {
    // app config dir
    fpath = GetPathRelativeToAppDirectory(path, appName);
    if (std::filesystem::exists(fpath)) {            // throwing form
        return fpath;
    }
    // app install dir
    fpath = GetPathRelativeToAppBundle(path);
    if (std::filesystem::exists(fpath)) {            // throwing form
        return fpath;
    }
    return "./" + std::string(path);
}
```

The throwing overload of `std::filesystem::exists(p)` raises
`filesystem_error` whenever the underlying `GetFileAttributesExW` returns
anything other than an ENOENT-equivalent. Probing
`D:/a/BattleShip/.../gamecontrollerdb.txt` on a machine whose `D:` drive
has no media returns `ERROR_NOT_READY` ("The device is not ready") — that
is **not** treated as "file doesn't exist," so the call throws.

The unwind crosses out of `osContInit` (no catch in the controller-init
path), `std::terminate` fires, and the spdlog async pool tears down on the
way out — yielding the user-visible `0xE06D7363` crash with no useful
traceback.

The crash hits at the controller stage rather than during the earlier
f3d.o2r bootstrap because the port-side `f3d.o2r` and `BattleShip.o2r`
loads already use a port-local `PortLocateFile` helper (issue #58, in
`port/port.cpp`) that calls the noexcept `exists(p, ec)` overload and
probes via `RealAppBundlePath()` — `gamecontrollerdb.txt` was the one
remaining caller routed through the upstream LUS helper because the
osContInit definition lives inside libultraship's own `os.cpp`, not the
port.

## Fix

Two hunks in `JRickey/libultraship#ssb64`:

**`src/ship/Context.cpp`** — switch `LocateFileAcrossAppDirs` to the
noexcept `exists(p, ec)` overload. `false` is the right answer for any
filesystem failure when probing a candidate path, and the throwing
overload had no benefit here. This single change fixes the crash for any
caller, not just `osContInit`.

```cpp
std::error_code ec;
…
if (std::filesystem::exists(fpath, ec)) { return fpath; }
```

`<filesystem>` and `<system_error>` were transitively pulled before; both
are now included explicitly so the noexcept overload's `error_code`
argument resolves cleanly.

**`src/libultraship/libultra/os.cpp`** — wrap the
`LocateFileAcrossAppDirs` + `SDL_GameControllerAddMappingsFromFile` block
in a try/catch. Loading `gamecontrollerdb.txt` is a quality-of-life
feature; failure to find or parse it should never crash the controller
thread. This is defense-in-depth in case any future path-helper change
re-introduces a throw on a hostile path. The catch logs via `SPDLOG_ERROR`
and falls through — `SDL_Init(SDL_INIT_GAMECONTROLLER)` and the
ControlDeck still come up with the SDL-builtin mappings.

The `CMAKE_INSTALL_PREFIX` issue itself (LUS embedding the build host's
runner path under `NON_PORTABLE`) is separate and tracked by the existing
port-side `RealAppBundlePath()` /`PortLocateFile` workaround in
`port/app_paths.cpp` — the project never intends to use the LUS bundle
path on Windows.

## Files

- `libultraship/src/ship/Context.cpp` — noexcept `exists(p, ec)` in
  `LocateFileAcrossAppDirs`; explicit `<filesystem>` / `<system_error>`
  includes.
- `libultraship/src/libultraship/libultra/os.cpp` — try/catch around
  `osContInit`'s gamecontrollerdb load.
- libultraship submodule pointer bumped to pick up both.

## Audit hooks

- Any libultraship API that probes the filesystem with
  `std::filesystem::exists(p)` (no `error_code`) is a latent crash on
  Windows where `GetFileAttributesExW` distinguishes ENOENT from other
  failures. Audit `Ship::Context`, `ResourceManager`, `ArchiveManager`
  for the same pattern.
- Any `osContInit`-class entry point that runs on a non-main thread and
  performs file I/O needs a catch-all so a throw doesn't kill the whole
  process before the GUI is up.
- The "CI bakes the runner path into a release artifact" failure mode
  applies to every NON_PORTABLE Windows packaging script; consider
  passing `-DCMAKE_INSTALL_PREFIX=.` (or omitting `NON_PORTABLE` entirely
  on Windows where the port already overrides via
  `RealAppBundlePath()`).

Class: **third-party path helper throws on non-ENOENT failure → unhandled
exception on user thread → terminate**. Same family as
`oscontpad_lus_sizeof_overrun_2026-04-24.md` (LUS code shipping in the
binary that the port doesn't have direct authority over until the
submodule bump lands).
