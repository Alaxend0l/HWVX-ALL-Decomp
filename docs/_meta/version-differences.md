# Global version differences — GC ↔ PS2 ↔ PC

All numbers below were computed from each project's own symbol/config files
(`HWVX-GC-Decomp/config/HOTWHE/symbols.txt`,
`HWVX-PS2-Decomp/symbol_addrs.txt`,
`HWVX-PC-Decomp/config/hotwheels/functions.json`) and are re-derivable.

## Code totals

| | GC (reference) | PS2 | PC |
| --- | --- | --- | --- |
| Named function symbols | 2,495 | 2,285 | 1,643 (all but 6 are `_func_XXXXXXXX` placeholders) |
| Sum of function `size:` fields | 812,892 B (~794 KiB) | 760,456 B (~742 KiB) | 481,039 B (~470 KiB) |
| Source units | 309 split `src/*.c` files | 1 module (`src/cod.c`) — naming/split pending | 165 units (`text_00401070`…) |
| Entry / base | DOL, `.init` at `0x80003100` | ELF `vram 0x00200000`, `_start` at `0x00200008` | PE image base `0x400000`, first unit `text_00401070` |

The PC build is ~40 % smaller in code than either console build; PC-only
libraries (OpenGL/GLUT, DirectX sound) partly make up for that with data and
imports, but the game code itself is a smaller footprint (fewer debug/printf
paths, different renderer).

## GC ↔ PS2 symbol-name overlap

The PS2 retail ELF is **not stripped** — `symbol_addrs.txt` carries the original
CodeWarrior symbol names (628 of the 2,285 are runtime/SDK: `_start`, `_dpfge`,
`sceGsResetGraph`, `kCopy*`, …). That makes name comparison with GC meaningful:

* **1,161 names appear in both** GC and PS2 symbol tables.
* 1,334 are GC-only; 1,124 are PS2-only.
* The **Action module is a clean example**: 48 `Action*` names are shared
  verbatim (`ActionAirboxChangeMode` … `Action_ReleaseAllActions`); GC-only adds
  `Action_Init` (PS2's ELF has no `Action_Init` symbol — the loop that zeroes the
  80-entry action table was inlined or absent), and PS2 contributes nothing extra.
* Naming convention is the same in both builds (mixed `ActionFoo` / `Action_Foo`
  styles come from the original sources, not the decomp), so PS2 docs can reuse
  GC names 1:1 once the `cod` segment is split.

## Shared names, different code

Only **107 of the 1,161 shared-name functions have identical byte sizes**; 1,054
differ (PPC vs MIPS codegen — a size match across ISAs is expected only for tiny
stubs). The largest deltas concentrate in the renderer, which is the strongest
evidence of a genuinely different implementation, not just a recompile:

| function | GC | PS2 | note |
| --- | --- | --- | --- |
| `Render_DrawLine2D` | 0x90 | 0x8c0 | PS2 ~10× bigger |
| `Render_ScreenFlash` | 0x22c | 0x9e4 | |
| `Render_DrawSprite` | 0x354 | 0x9c4 | |
| `Render_DrawRectangles(Window)` | 0x1c4 | 0x830 | |
| `Render_EndScene` | 0xae0 | 0x5c0 | PS2 *smaller* |
| `LightManager_LightThing` | 0xef4 | 0x1658 | |
| `Camera_UpdatePosition` | 0xe44 | 0x84c | PS2 smaller |
| `CarA_ComputeSuspensionForces` | 0x1310 | 0x18ac | |

Gameplay/logic deltas are much smaller: the whole `Action*` family sits within
±20 % of GC sizes, e.g. `ActionVehicleChangeMode` 0x3bc → 0x45c,
`ActionMusicChange` 0x2ec → 0x424, `ActionRoundComplete` 0x28 → 0x20.
Outliers in that family (`Action_Create` 0x20 → 0x4c, `Action_Destroy` 4 → 8,
`Action_AddAction` 0x148 → 0xb0, `Action_ReleaseAllActions` 0x30 → 0x14) are
**not** logic differences — verified against `asm/nonmatchings/cod/`:
`Action_Heartbeat` and `Action_AddAction` are the same algorithm at the same
record field offsets (`type`@0x0, `data`@0x4, `count`@0x10, `delayInit`@0xC,
`delay`@0x14; 80 slots × 4 at `D_518D50` ≡ `m_Actions_800DBC80`), the size
deltas being call-sequence/codegen artifacts (`Action_Destroy` 4→8 is the
MIPS minimum empty frame). The one real registry difference: **PS2 has no
`Action_Init`** — its GC body (zero the 0x140-byte table) is folded into
`Action_Create`, while GC's `Action_Create` is a stub forwarding to
`fn_800957D4`. See `docs/Action.c/`.

## Platform-only symbol families (what each version carries that the others don't)

* **GC-only (1,334)**: `GX*`/`Gfx*` display-list paths, `VDevice_*` (GC video
  output), `Render_*_` display-list builders (`DLBuild*`), `ExPPC_*` and
  `__cxa`/`TRK` names (PPC C++/exception runtime), `EXI2*` stubs, `OS*`/`dvd*`
  (Nintendo SDK).
* **PS2-only (1,124)**: `sce*` PS2 SDK calls (`sceGsResetGraph`, `sceSif*`),
  kernel helpers (`_dpf*`, `kCopy*`), `TextureManager_*` (VU/texturing path),
  `USBFile_*`, `Memcard_*`, `eh*`/`setup*` startup glue.
* **PC resolved names (6)**: `glEnd`, `gluSphere`, `gluNewQuadric`,
  `gluBuild2DMipmaps`, `gluScaleImage`, `DSOUND_11` — i.e. the PC port links
  **OpenGL + GLUT** and a **DirectX sound** thunk instead of the console renderers.
  Everything else in `functions.json` is still an address-named placeholder —
  the PC build shipped **without debug symbols**, so those names are ours to
  recover (see the working plan below).

## Data / ABI differences to keep in docs

* **PS2** uses gp-relative small data: `_gp = 0x002D7EF0`; the best-scoring
  attempts in `progress/decompdecomp.json` used `mwcps2 -O2 -sdatathreshold 8`,
  so `_s*`-style symbols live in `.sdata`/`.sbss` and function docs must say when
  an access is gp-relative.
* **GC** splits SDK data into `.init`, `.s2.rdata`, `.s2.bss` etc. (dtk
  `symbols.txt` section prefixes); globals are written `name = .section:0xADDR`.
* **PC** is a PE with absolute relocations (`_D_004806E0`-style data refs in
  `functions.json`, `/Oy-` frame-pointer flags on 4 units) — no gp trickery,
  different struct-padding expectations.

## Working plan (order of attack)

1. Split PS2 `cod` into GC-mirrored modules once names exist in
   `symbol_addrs.txt` — the 1,161-name overlap gives the boundaries directly.
2. Port GC file names to PS2 `splat` subsegments; then PS2 per-function C lives
   beside GC's, and docs go under `docs/<file>/<Function>.md` for both.
3. PC: rename `text_*` units toward GC files using the relocs/data refs; docs
   start with the 6 GL/DX entries and the obvious physics modules.
