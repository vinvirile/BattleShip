# Issue #72 — Yoshi tongue 1-frame pink-flash — Handoff (2026-05-02)

## Symptom (per user)

During the Yoshi segment of the opening attract intro, on the port,
there is a single frame where the entire framebuffer renders pink.
GH issue title: "Camera blocked by something for a few frames during
intro sequence (Yoshi segment)". User clarified later: it's a 1-frame
full-framebuffer pink draw, not multi-frame occlusion. Reference
correct behaviour: RMG + GLideN64 + Azimer-HLE + Pure Interpreter —
no flash, no occlusion.

User-reported platform: Windows 10 + DX11 debug build of `a3e242fd4afe`.

## Investigation status — UNRESOLVED

A first-pass subagent ran on this branch and produced an earlier
handoff that hypothesised an "attract chain silently skips Link and
Kirby" bug. **That hypothesis is wrong.** The agent confused scene
IDs: in `src/sc/scdef.h` enum order `33 = Fox, 34 = Link, 35 = Yoshi,
36 = Pikachu, 37 = Kirby`, but the agent assumed numerical order
matched the in-game intro order. The port's actual scene-entry log
sequence `30 → 31 → 34 → 32 → 35 → 37 → 33 → 36 → 38` decodes to
`Mario → DK → Link → Samus → Yoshi → Kirby → Fox → Pikachu → Run`,
which IS the canonical SSB64 intro order. **No scene is being
skipped.**

## What was actually checked

- Captured 2086-frame `port_trace.gbi` of the natural attract chain
  via `SSB64_GBI_TRACE=1` (size 473 MB).
- Searched for full-framebuffer `G_FILLRECT (0,0)-(320,240)` —
  appears only in frames **1093-1132** (40-frame range during
  `mvOpeningRoom` transition explosion at scene 28's tic ~1040; the
  Outline silhouette's intentional N64 colour-image-redirect-to-Z
  fill, already documented and partially handled in
  `docs/bugs/intro_explosion_overlay_layering_2026-04-26.md`). Yoshi
  scene's frames are well past that window — **no port-side full-FB
  rect emission is responsible** for any later flash.
- Searched all `src/`, `port/`, `libultraship/src/fast/` for literal
  pink/magenta colour constants. Found:
  - `mvOpeningYoshiPosedWallpaperProcDisplay` peach `0xFFBE5A`
    `gDPSetPrimColor` — but the FillRect after it is panel-clipped
    `(10,150)-(310,230)`, NOT full-screen. Innocent.
  - `stb_image_write.h` background `255,0,255` — PNG composition
    only, irrelevant to runtime.
  - No magenta debug-fallback pixels in `gfx_metal.cpp` /
    `gfx_direct3d11.cpp` clear paths.
- `mvopeningyoshi.c` reviewed end-to-end: cleanly resets
  `TotalTimeTics`, clean scene transitions, no risky redirect
  patterns.

## Hypotheses still on the table

1. **Renderer-side, NOT visible in the GBI trace.** A texture rect
   with bad scissor extents, a Metal/DX11 backend transient
   render-target rebind, or a one-frame texture-cache miss that
   binds an uninitialized texture (cf. `project_whispy_curr_eq_next`
   — TEXEL1 unbound on a CC that referenced it). The GBI trace would
   not reveal these because the GBI commands are correct; the bug is
   in how the backend interprets them in a specific Metal state.

2. **Recent framebuffer-capture work introduced the regression.**
   Commits `24bc4e6` (Pin LUS off-screen rendering when stage-clear
   frozen wallpaper is on) + `ee9204b` + the LUS `bb6dae5` ForceRenderToFb
   override toggle FB targets at PortInit based on a CVar. If
   `sForceRenderToFbDesired = true` toggles the render target on a
   specific frame transition, a one-frame stale framebuffer texture
   showing through is plausible. **Test:** disable the
   StageClearFrozenWallpaper CVar (or set the env var that controls
   it) and re-run; if the pink flash disappears, the FB-capture path
   is the culprit. CVar lives in `port/port.cpp` at end of `PortInit`.

3. **DX11-specific backend path.** The user's report is on Windows
   DX11. Mac/Metal may not reproduce. Testing on this Mac is unlikely
   to surface the bug; need a Windows reproducer or remote DX11
   instrumentation (`SSB64_GFX_DEFER_VI`, `SSB64_RCP_*` env vars in
   `port/gameloop.cpp` + `port/stubs/n64_stubs.c` are worth
   experimenting with).

## What did NOT cause it (ruled out)

- Magenta literal in shader / clear / debug fallback (none found).
- Full-FB `G_FILLRECT` in the trace (only the OpeningRoom transition
  uses one, and it's already guarded against landing on the FB by
  LUS commit `0148b85`).
- Scene skip / scene mis-dispatch (false positive from the prior
  agent — the chain order is correct).
- TEXEL1 unbound regression of the Whispy class (the LUS fix from
  `project_whispy_curr_eq_next` was specifically zero-init
  `mCurrentTextureIds[1]` + 1x1 black fallback texture; verify it's
  still in place by grepping for `mCurrentTextureIds` and the
  fallback texture creation).

## Next-session prompt

1. View the user's two attached videos in GH issue #72 — the 1-line
   visual summary (colour vs occlusion vs both, what exact frame
   timing relative to the tongue extension).
2. If colour: instrument LUS `gfx_metal.cpp` /`gfx_direct3d11.cpp`
   `BeginRenderPass` to log every clear-color invocation — capture a
   3-second window over Yoshi tic 0-14, find any non-black clear-color
   value.
3. If occlusion / "blocked by something": instrument the GObj draw
   chain at `gcDrawAll` to log each gobj's `display` callback fire
   per-frame during Yoshi tic 0-14, find the unexpected gobj.
4. Toggle `gStageClearFrozenWallpaper` CVar off (port menu or
   `port/configvar.cpp`) and rerun. If the flash disappears, the bug
   is in the FB-capture force-render-to-FB path.
5. **The 60-second visual check is the unblocker.** Pixel-level visual
   evidence is needed — Mac/Metal rebuilds without that signal are
   throwing darts.

## Files referenced

- Trace: `<worktree>/build/debug_traces/port_trace.gbi` (473 MB,
  2086 frames)
- `src/mv/mvopening/mvopeningyoshi.c`
- `port/bridge/framebuffer_capture.{h,cpp}`
- `libultraship/src/fast/Fast3dWindow.cpp`
- `libultraship/src/fast/backends/gfx_metal.cpp`
- User-attached log + videos at <https://github.com/JRickey/BattleShip/issues/72>

## Update — 2026-05-02 evening session

**User clarified the symptom:** not full-FB pink flash. The actual bug is
**yellow vertex geometry blocking part of the camera at the top of the
screen, only during Yoshi's tongue animation.**

**User also noted:** the bug reproduces on inaccurate emulators
(mupen64plus + dynarec / HLE) but NOT on Rosalie's Mupen GUI with the
Pure Interpreter CPU + Azimer HLE RSP combo. **This is a CPU-accuracy
class bug** — i.e., the same family as our LP64 / pointer-arithmetic
regressions where x64/arm64 native-recompile semantics diverge from
N64 MIPS semantics.

### New hypothesis

The captured fighter (Pikachu, the yellow one Yoshi grabs in the
intro) is positioned via `ftCommonCapturePulledRotateScale`
(`src/ft/ftcommon/ftcommoncapturepulled.c:11`) which reads:

```c
DObj *joint = DObjGetStruct(fighter_gobj)->child;
func_ovl0_800C9A38(mtx, capture_fp->joints[capture_fp->attr->joint_itemheavy_id]);
func_ovl2_800EDA0C(mtx, rotate);
this_pos->x = (-joint->translate.vec.f.x * scale.x);
…
gmCollisionGetWorldPosition(mtx, this_pos);
```

`func_ovl0_800C9A38` (`src/lb/lbcommon.c:1667`) walks
`ftGetParts(dobj) → parts->mtx_translate` to build the world-space
matrix the captured fighter is pinned to.

The same family of bug already shipped a fix:
`fighter_slope_contour_lp64_alias` (memory note + bug doc) — fighter
foot IK was reading a cached transform through an
`FTParts*`-as-`DObj*` alias that LP64 widening broke. The captured-
fighter positioning code path traverses similar FTParts-side data and
could have the same class of issue: an alias / cast / pointer-stride
that's correct under N64 32-bit pointers but wrong under LP64.

### Visual evidence

Saved at `docs/issue_72_caps/cap_0{2-7}.png` — direct boot via
`SSB64_START_SCENE=35 ./BattleShip`, screencap'd at ~32s wall-clock
(after the 1875-tic FuncStart wait). On Mac Metal the captures show
roughly normal frames (cap_03 shows the tongue grab moment with
Pikachu visible at the upper-left of Yoshi's snout) — the Mac Metal
build may not exhibit the bug as severely as Windows DX11, OR the
visual difference is subtle in stills and only obvious in motion.

### Suggested next steps (revised)

1. **Confirm on Windows DX11 first.** Run the same direct-boot test
   on the user's machine, capture the broken frame.
2. **Audit FTParts/DObj alias paths** in `ftcommoncapture.c`,
   `ftcommoncapturepulled.c`, `ftcommoncaptureyoshi.c`. Apply the
   same #ifdef PORT explicit-FTParts-row-read pattern from
   `fighter_slope_contour_lp64_alias`.
3. **Instrument `func_ovl0_800C9A38`** to log
   `parts->mtx_translate` row 0 when entered for a captured fighter
   in the Yoshi tongue context. Compare against expected joint
   matrix on RMG/PureInterp.
4. **Check the FTAttributes file load** for fkind=Yoshi (fkind=6).
   `joint_itemheavy_id` is at struct offset 0x334 — verify pass1
   BSWAP + any pass2 fixup for that field reads correctly.

## Update — 2026-05-02 second evening session

User confirmed bug is identical on Mac and Windows (rules out backend
difference). Bug does not reproduce on RMG with Pure Interpreter +
Azimer HLE. Confirmed CPU-accuracy class bug.

### Empirical findings against the live port trace + screen captures

1. **The captured-fighter code path doesn't fire during Yoshi's intro.**
   Instrumented `ftCommonCaptureYoshiProcPhysics:76` and
   `ftCommonCapturePulledRotateScale` with port_log (gated by
   `SSB64_TRACE_INTRO_ANIM=1`). Built, ran scene 35 for a full Yoshi
   visible-scene window — zero log entries fire. Yoshi's
   `dMVOpeningYoshiKeyEvents` triggers Z+A grab against empty space
   (the demo has no opponent), so no `capture_gobj` is ever set.
   **The "yellow geometry" is NOT a captured Pikachu being
   mispositioned.** Reverted instrumentation.

2. **No `G_SETPRIMCOLOR FFFF00FF` (pure yellow) during Yoshi's frames.**
   Captured 177-frame `port_trace.gbi` (`SSB64_GBI_TRACE=1`,
   `SSB64_START_SCENE=35` direct boot, ~38s wall-clock through
   FuncStart wait + visible Yoshi scene + carry-through to Kirby/Fox/
   Pikachu chain). Per-frame cmd-count bucketing places Yoshi's
   visible scene at trace frames **14-58**. Yellow `G_SETPRIMCOLOR
   FFFF00FF` appears only at trace frames **132-176** — Fox+Pikachu
   segments, not Yoshi. The unique-color sweep (`grep -oE
   "G_SETPRIMCOLOR..." | sort -u`) shows the only "warm" colors active
   during Yoshi are `FFBE5A` (peach posed-wallpaper),
   `FF8000` (orange), `FF2600` (red — likely the tongue), and
   `FFA4B8` (pink). No pure-yellow PRIM in the Yoshi window.

3. **macOS screen capture with `ffmpeg -f avfoundation -i 2`
   60fps** → 150 frames over 2.5s, exported to PNG. Saved earlier
   stills at `docs/issue_72_caps/cap_03.png` etc. Visually inspecting
   the Yoshi tongue moment (frames 25-40 of the recording, mapping to
   trace frames ~25-40 of the visible scene) does NOT clearly show
   "yellow geometry blocking the top of the camera" the way the user
   describes. Either the bug is much subtler at the actual resolution
   the port renders in our Mac window, or there's a specific frame
   between samples that exhibits the symptom.

### What this rules out

- **Captured-fighter rendering path** (no captured fighter exists).
- **Yellow `G_SETPRIMCOLOR`** drawing the bug (no FFFF00FF during
  Yoshi).
- **`G_FILLRECT (0,0)-(320,240)` full-FB rect** (none in trace
  frames 14-58 — only the OpeningRoom transition uses one earlier
  in the natural attract chain).
- **Magenta debug fallback** in libultraship Metal/D3D11 backends
  (none found in code review).

### What remains plausible

1. **Vertex-color yellow on a specific mesh.** The "yellow geometry"
   may come from per-vertex color in `G_VTX` data (loaded from RAM,
   not visible in GBI command stream). A specific Yoshi DObj or
   accessory's vertex stream might encode yellow vertex colors that
   on N64 are correctly clipped/positioned, but on LP64-recompile the
   skeleton matrix or vertex transform is wrong → yellow vertices
   appear at unintended screen position.
2. **Yellow texture pixels through a CI palette.** The CI4/CI8 LUT
   for some Yoshi-era texture may contain yellow indices that on N64
   are referenced by a DObj that's clipped, but on LP64 a stride bug
   causes the wrong DObj to draw.
3. **Specific Yoshi figatree event** with unusual scaling /
   positioning. The grab animation has tongue-extension events that
   move bones to extreme positions — if any LP64 quirk doubles or
   misaligns those values, a tongue mesh segment could land near
   y=0 (top of screen).

### Tooling status

- `port_trace.gbi` capture: works.
- `mupen64plus-rsp-trace.dylib` (emu side): rebuilt, not used here
  yet — would need a side-by-side capture against an emu that DOES
  show the bug (so `mupen64plus` HLE/dynarec would show the bug per
  user, not RMG/Pure Interpreter; that diff would localise the GBI
  command).
- macOS screen capture: works at 60fps via ffmpeg avfoundation.
- Built-in `portFastCaptureBackbufferPNG`: BROKEN on Mac Metal
  (returns 0 every call). Worth fixing separately.

### Suggested next-session prompt

1. **Run mupen64plus + our RSP trace plugin** through the natural
   attract chain to capture an emu-side `emu_trace.gbi`. Then run
   `gbi_diff.py port.gbi emu.gbi --frame-range 14-58
   --ignore-addresses` to find Yoshi-segment GBI divergences. The
   port and `mupen64plus + dynarec/HLE` should both have the bug;
   diff against RMG would be ideal but RMG doesn't bundle our trace
   plugin natively (needs cross-compilation). The mupen64plus HLE
   trace gives a baseline of "what the game's CPU emits with HLE-RSP
   producing identical broken output to ours" — divergences would
   be Fast3D-specific.
2. **Or:** instrument `func_ovl0_800C9A38` /
   `ftPhysicsGetRootMotionJoint` walks for Yoshi's 60-tic visible
   scene to dump every joint matrix translate. Compare against the
   tongue-grab figatree event data. Wildly out-of-range values would
   pinpoint the LP64 quirk.
3. **Or:** ask the user for a Windows-side annotated screenshot of
   the bug frame so we know exactly which screen-space pixel range
   the yellow geometry occupies. That + the GBI trace makes
   localisation tractable.

### Files

- `port_trace.gbi`: 177 frames, 12.4 MB at
  `<worktree>/build/debug_traces/port_trace.gbi`
- ffmpeg recording: `/tmp/yoshi_video/scene.mov` (~95 MB) +
  150 PNG frames
- Mac Metal still captures: `<worktree>/docs/issue_72_caps/cap_0{2-7}.png`
