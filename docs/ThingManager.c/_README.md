# `ThingManager.c` — thing pools, global active list, slave lists

This module owns the game's **thing object pool**: every spawnable object
(projectiles, items, effects, racks, cars, creatures, sound sequences…) lives
in one of 18 fixed pools with a per-type free list and a single global active
list. It also implements the slave-list machinery (slaves are created attached
to a host's private child list and only joined to the active list when
"emancipated") and the per-frame `Heartbeat`/`MoveThings`/`Render` loops.

The `DARYL` name component in most function names is the game's internal
label for this manager (kept verbatim from the GC symbols).

## PC unit layout

Unlike `Action.c`, the PC build does **not** keep this module in one object
unit; the MSVC units are split around the GC file boundary and mixed with
`Thing.c` functions:

| PC unit | Range | ThingManager.c content |
| --- | --- | --- |
| `text_0044A0D0` | `0x0044A0D0`–`0x0044AD00` | list-op cluster `0x0044A790`–`0x0044AA50` (tail of the unit); the rest is `Thing.c` (e.g. `Thing_ProcessEndEvent` at `0x0044A430`) |
| `text_0044D050` | — | `ThingManager_DARYL_CreateThing` `0x0044D0B0`, `ThingManager_DARYL_CreateSlave` `0x0044D0F0` |

The list cluster starts at `0x0044A790`; everything before it in the unit is
`Thing.c`. One small foreign function sits inside the cluster window:
`0x0044A770` (`return x->[0x20] == v ? x->[0x24] : x->[0x20]`) is **not** a
ThingManager list op — treat it as an unmapped foreign one-liner.

## Shared `ThingHeader` layout

All three builds agree on these offsets (verified from the linked-list ops,
PC disassembly field offsets, and PS2 `.s` load/store offsets):

| Offset | Field | Notes |
| --- | --- | --- |
| `0x40` | `f32[4]` | bounding-sphere centre (used by `Render` cull) |
| `0x70` | `f32` | bounding-sphere radius |
| `0x74` | `u32 flags` | see flag table below |
| `0x80` | `s16` | grid cell index (`cmp word ptr [ebp+0x80],0` in PC `EmancipateSlaves`) |
| `0x82` | `u8 slaveId` | trigger id of a slave (written by `CreateSlave`) |
| `0x83` | `u8 slavePlayer` | slave trigger filter; `0xEF` = emancipate marker |
| `0x88` | `void *` | private child/slave list head (recursed by `ReleaseAndRemoveAll`) |
| `0x8C` | `void *vtable` | `[0]` = `Init(this, params)`, `[1]` = `Release(this)` |
| `0xA0` | `struct ThingHeader *prev` | intrusive doubly-linked list |
| `0xA4` | `struct ThingHeader *next` | |
| `0xA8` | `u32 type` | free-list index (0–17). **PS2 reads/compares it as a byte** |

Flags (observed masks): `0x8` in-world/grid, `0x1000` on free list,
`0x200` released, `0x200000` released-and-pending-free (the sweep's marker,
set with `0x200` and `2` by `Thing_Release` at `HWVX-GC-Decomp/src/Thing.c:80`),
`0x20000000` slave (set by `CreateSlave`). `Render` skips things with
`flags & 0x200200`.

## The pool struct — identical layout on all three platforms

`HWVX-GC-Decomp/src/ThingManager.c:47`–`69` types `g_thingManager_DARYL`
(GC `.bss 0x8022F350`, size `0x7ABC0`) as 18 static thing arrays followed by:

```c
    void *activeList;                   /* 0x7AB70 */
    void *freeList[19];                 /* 0x7AB74 */
```

The same tail offsets are confirmed on PS2 and PC, which is the strongest
module fingerprint available for this unit:

| Platform | Manager base | `activeList` | `freeList[]` | Evidence |
| --- | --- | --- | --- | --- |
| GC | `0x8022F350` (`.bss`) | `+0x7AB70` | `+0x7AB74` | symbol `g_thingManager_DARYL` |
| PS2 | `0x0066A030` | `D_006E4BA0` = base+`0x7AB70` | `D_006E4BA4` | `ThingManager_DARYL_CreateThing.s`: `lui %hi(D_66A030)` + `ori $3,$3,0x7AB70` @`0x002AD63C`–`0x002AD648` |
| PC | `0x004BFE60` (derived: `0x53A9D0 − 0x7AB70`; no symbol) | `0x0053A9D0` | `0x0053A9D4` | `CreateThing` `lea ecx,[eax*4+0x53a9d4]` @`0x0044D0B4`, `push 0x53a9d0` @`0x0044D0CC` |

Type indices: `0` projectiles(140), `1` eventSeqs(192), `2` emitters(160),
`3` destructibles(500), `4` items(96), `5` attaches(192), `6` debris(128),
`7` creature(1), `8` wildCardRacks(16), `9` cars(8), `10` physicalObjects(100),
`11` looker(1), `12` tracers(32), `13` pod(1), `14` drone(1), `15` motions(56),
`16` soundSeqs(128), `17` radarBlips(64). No platform bounds-checks
`freeList[type]`.

## Lifecycle

* `CreateThing(type, params)` = pop head of `freeList[type]` → **append to
  `activeList`** → `vtable[0](thing, params)`. Init runs *after* the thing is
  on the active list.
* `CreateSlave(type, params, id)` = pop → `vtable[0]` → `slaveId = id`,
  `flags |= 0x20000000`. **No active-list append** — a slave hangs off its
  host's `+0x88` child list until `ThingManagerList_EmancipateSlaves` joins it
  to the active list.
* `Thing.c`'s `Thing_Release` marks `flags |= 0x200 | 0x200000`; the sweep
  (`RemoveReleasedThings`) then calls `vtable[1]`, unlinks, re-appends to
  `freeList[thing->type]` and sets `flags = (flags & ~0x200000) | 0x1000`.

## Function map

GC from `HWVX-GC-Decomp/config/HOTWHE/symbols.txt`; PS2 from
`HWVX-PS2-Decomp/symbol_addrs.txt` (bodies under
`HWVX-PS2-Decomp/asm/nonmatchings/cod/`); PC verified by disassembly
(sizes = Ghidra extents incl. `int3` padding). `—` = not yet identified.

| Function | GC addr / size | PS2 addr / size | PC addr / extent | Doc |
| --- | --- | --- | --- | --- |
| `ThingManagerList_Append` | `0x80053F78` `0x28` | `0x002B1910` `0x24` | `0x0044A790` `0x30` | [→](ThingManagerList_Append.md) |
| `ThingManagerList_Remove` | `0x80053FA0` `0x48` | `0x002B18D0` `0x40` | `0x0044A7C0` `0x50` | [→](ThingManagerList_Remove.md) |
| — (PC-only pop helper) | inline | inline | `0x0044A810` `0x30` | [→](ThingManagerList_Pop.md) |
| `ThingManagerList_GetByType` | `0x80053FE8` `0x28` | `0x002B1890` `0x3C` | `0x0044A840` `0x30` | [→](ThingManagerList_GetByType.md) |
| `ThingManagerList_Get` | `0x80054010` `0x2C` | `0x002B1850` `0x34` | `0x0044A870` `0x30` | batch 2 |
| `ThingManagerList_ReleaseSlavesBySlaveType` | `0x8005403C` `0x110` | `0x002B1740` `0x10C` | `0x0044A8A0` `0x70` | batch 2 |
| `ThingManagerList_RemoveReleasedThings` | `0x8005414C` `0xF8` | `0x002B1610` `0x12C` | `0x0044A910` `0x60` | batch 2 |
| `ThingManagerList_ReleaseAndRemoveAll` | `0x80054244` `0x228` | (no symbol, see note) | `0x0044A970` `0x20` | batch 2 |
| `ThingManagerList_ReleaseAll` | `0x8005446C` `0x4C` | `0x002B15C0` `0x48` | `0x0044A990` `0x30` | batch 2 |
| `ThingManagerList_EmancipateSlaves` | `0x800544B8` `0x158` | `0x002B1480` `0x13C` | `0x0044A9C0` `0x90` | batch 2 |
| `ThingManagerList_TriggerSlaves` | `0x80054964` `0x6B0` | `0x002B04C0` `0xA94` | — | |
| `ThingManagerList_InitAllSlavesNow` | `0x80055014` `0x170` | `0x002B0380` `0x140` | — | |
| `ThingManager_DARYL_CreateMain` | `0x80055184` `0xF3C` | `0x002AF7A0` `0xBD4` | — | |
| `ThingManager_DARYL_Create` | `0x800560C0` `0x44` | `0x002AF750` `0x50` | — | |
| `ThingManager_DARYL_Destroy` | `0x80056104` `0xCC` | `0x002AF630` `0x120` | — | |
| `ThingManager_DARYL_Init` | `0x800561D0` `0x24` | `0x002AF5C0` `0x68` | — | |
| `ThingManager_DARYL_Release` | `0x800561F4` `0x4C` | `0x002AF490` `0x12C` | — | |
| `ThingManager_MoveLookerOnly` | `0x80056240` `0xAC` | (no symbol) | — | |
| `ThingManager_DARYL_Heartbeat` | `0x800562EC` `0x34` | `0x002AF3D0` `0xBC` | — | |
| `ThingManager_MoveThings_DARYL_SUBTICK` | `0x80056320` `0x185C` | `0x002AD720` `0x1CAC` | — | |
| `ThingManager_DARYL_Render` | `0x80057B7C` `0x9C` | `0x002AD690` `0x8C` | — | |
| `ThingManager_DARYL_CreateThing` | `0x80057C18` `0xC8` | `0x002AD5E0` `0xB0` | `0x0044D0B0` `0x40` | [→](ThingManager_DARYL_CreateThing.md) |
| `ThingManager_DARYL_CreateSlave` | `0x80057CE0` `0xB4` | `0x002AD540` `0x98` | `0x0044D0F0` `0x50` | [→](ThingManager_DARYL_CreateSlave.md) |
| `ThingManager_GetActiveCars` | `0x800585B8` `0xC4` | `0x002ACA70` `0x68` | — | |
| `ThingManager_GetActiveItems` | `0x8005867C` `0x10C` | `0x002ACA00` `0x68` | — | |
| `ThingManager_GetActiveSoundSequences` | `0x80058788` `0xC4` | `0x002AC990` `0x68` | — | |
| `ThingManager_Invalidate` | `0x80058DCC` `0x544` | `0x002ABD60` `0x744` | — | |

Also in the GC source but without link symbols: `ThingManager_InitFreeThing` /
`ThingManager_InitPool` (`static inline`, source lines 71/91),
`SharedSlaveHolder_Release` (:268), `SharedSlaveHolder_Get` (:292),
`Thing_DrawSharedSlaves` (:312).

## PS2 symbol gaps

* **No `ThingManagerList_ReleaseAndRemoveAll` symbol.** `ReleaseAll` ends at
  `0x002B1608` and `RemoveReleasedThings` starts at `0x002B1610` — an 8-byte
  slot exactly where GC source order (PS2 lays out in reversed source order)
  places `ReleaseAndRemoveAll`. It is plausibly a two-instruction tail-call
  stub (`jal ReleaseAll; j RemoveReleasedThings`), mirroring the PC shape, but
  this is **unverified**.
* **`ThingManager_MoveLookerOnly` has no PS2 symbol** — removed or inlined in
  the PS2 build.
* `ThingManagerList_RemoveReleasedThings.s` recurses into *itself*
  (`jal ThingManagerList_RemoveReleasedThings` @`0x002B1670`), while GC's
  `ThingManager.c:174` recurses into `ThingManagerList_ReleaseAndRemoveAll`
  (which runs `ReleaseAll` on the child list first). Potential behavioural
  divergence — to be confirmed in batch 2.
