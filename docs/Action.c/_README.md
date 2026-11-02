# `Action.c` — the in-game "action" / trigger system

GC reference file: `HWVX-GC-Decomp/src/Action.c` (846 lines, 29 functions:
28 addressable + 1 `static inline` helper).

## What the module is

The game drives scripted one-shot effects through a small registry:

* **Action records** — GC layout seen in the decomp is 6 dwords at stride
  `0x18`: `type`, `data` (pointer to the per-type argument array), two unused
  dwords, `delayInit`, `count`, `delay` (see `Action.c:112`-`119`).
* **A fixed 80-slot table** `m_Actions_800DBC80[80]` (GC) of record pointers —
  the same "max 80 simultaneous actions" limit appears in the game's own debug
  string in `Action_AddAction` (`Action.c:130`).
* **A dispatch table** `pActionFuncArray_800CF500[]` indexed by
  `record->type`, whose handlers are the `Action*ChangeMode`-style functions —
  some defined in *this* file, most in the subsystem that owns the thing being
  changed (`src/Thing.c`, `src/World.c`, `src/InterfaceManager.c`,
  `src/Emitter.c`, `src/AudioManager.c`, `src/CameraManager.c`, …).
* **`Action_Heartbeat`** advances every record's `delay` by `g_iDeltaTimeMS`
  and fires a record once its `delay` reaches ≤ 0; a record with type
  `0x14` survives the call unless its data array's `[1]` slot is `NULL`.

The `Action_*` (underscored) half of the file is that registry itself — create,
destroy, init, heartbeat, release, add. The `ActionCamelCase` half is a set of
handlers that happen to live in this file (vehicle/AI/creature/item/
interface/code-variable related).

## Functions

GC address/size from `config/HOTWHE/symbols.txt`; PS2 address/size from
`HWVX-PS2-Decomp/symbol_addrs.txt` (PS2 retail ELF is not stripped, so the
same names exist). PS2 bodies for a few of these also exist as m2c drafts in
`HWVX-PS2-Decomp/progress/best/`.

PC addresses/sizes are from disassembly of module unit `text_00401090`
(`0x00401090`–`0x00401FC8` in `orig/hotwheels.exe`), identified as this module
by the game's own "max number of actions" debug string at `0x0047A0F0`, the
80-slot `m_Actions` table at `0x004806E0`, and the action dispatch table at
`0x0047A040`; per-function evidence is in the individual docs. PC sizes are
function extents (code + embedded jump tables + alignment padding). A `—`
means the handler is a trivial one-liner MSVC largely inlined (not individually
pinned down in the dump); `ActionPrintDebugString` is absent from the PC unit
entirely. Two unit entries are still unmapped: `0x004011A0` (0x100 bytes, a
car-command switch) and the tail `0x00401F40`–`0x00401FC8`.

| function | GC `0x` | GC size | PS2 `0x` | PS2 size | PC `0x` | PC size |
| --- | --- | --- | --- | --- | --- | --- |
| `Action_GetCarPlayer` | — (static inline) | — | — | — | — (inlined) | — |
| `ActionCameraSetTargets` | 800057e0 | 0x24 | 24f510 | 0x20 | — | — |
| `ActionRoundComplete` | 80005804 | 0x28 | 24f4f0 | 0x20 | — | — |
| `ActionResetGameRound` | 8000582c | 0x48 | 24f4c0 | 0x2c | — | — |
| `ActionSetPlayerOrder` | 80005874 | 0x28 | 24f4a0 | 0x20 | — | — |
| `ActionSpeakerChangeMode` | 8000589c | 0x8 | 24f490 | 0x8 | — | — |
| `ActionVehicleAISetNamedPath` | 800058a4 | 0x8 | 24f480 | 0x8 | — | — |
| `ActionSetPlayerVehicle` | 800058ac | 0x8 | 24f470 | 0x8 | — | — |
| `ActionVehicleRemoveCreature` | 800058b4 | 0x8 | 24f460 | 0x8 | — | — |
| `ActionScriptLightChangeMode` | 800058bc | 0x8 | 24f450 | 0x8 | — | — |
| `Action_Create` | 800058c4 | 0x20 | 24f400 | 0x4c | — | — |
| `Action_Destroy` | 800058e4 | 0x4 | 24f3f0 | 0x8 | — | — |
| `Action_Init` | 800058e8 | 0xc0 | — | — (no PS2 symbol) | 4010a0 | 0x20 |
| `Action_Heartbeat` | 800059a8 | 0xc4 | 24f320 | 0xd0 | 4010c0 | 0x80 |
| `Action_ReleaseAllActions` | 80005a6c | 0x30 | 24f300 | 0x14 | 401120 | 0x20 |
| `Action_AddAction` | 80005a9c | 0x148 | 24f250 | 0xb0 | 401140 | 0x60 |
| `ActionPrintDebugString` | 80005be4 | 0x158 | 24f0f0 | 0x154 | — | — (absent) |
| `ActionLoadInterfacePack` | 80005d3c | 0x44 | 24f0b0 | 0x34 | 4012a0 | 0x30 |
| `ActionUnloadInterfacePack` | 80005d80 | 0x34 | 24f070 | 0x34 | 4012d0 | 0x20 |
| `ActionCodeVariableModifyValue` | 80005db4 | 0x244 | 24edc0 | 0x2a4 | 4012f0 | 0x230 |
| `ActionMissionObjChangeMode` | 80005ff8 | 0x38 | 24ed80 | 0x40 | 401520 | 0x30 |
| `ActionVehicleChangeMode` | 80006030 | 0x3bc | 24e920 | 0x45c | 401550 | 0x370 |
| `ActionNavPointChangeMode` | 800063ec | 0x34 | 24e8e0 | 0x38 | 4018c0 | 0x30 |
| `ActionPauseTACSystem` | 80006420 | 0x78 | 24e860 | 0x80 | 4018f0 | 0x50 |
| `ActionSwapAIScript` | 80006498 | 0x104 | 24e740 | 0x118 | 401940 | 0xc0 |
| `ActionVehicleRemoveItem` | 8000659c | 0xcc | 24e640 | 0xfc | 401a00 | 0x90 |
| `ActionVehicleGiveItem` | 80006668 | 0x39c | 24e280 | 0x3bc | 401a90 | 0x360 |
| `ActionPlaySoundSequence` | 80006a04 | 0xdc | 24e1a0 | 0xd4 | 401df0 | 0x80 |
| `ActionCreatureChangeMode` | 80006ae0 | 0x120 | 24e090 | 0x104 | 401e70 | 0xd0 |

Note the PS2 addresses run **descending** in the same call-graph order as GC's
ascending ones — the module's link order is reversed but identical, which is
useful for splitting PS2's `cod` segment.

## Not in this file, but dispatched by it

The other ~19 `Action*ChangeMode` handlers (`ActionAirboxChangeMode`,
`ActionTimerChangeValue`, `ActionHUDChangeMode`, `ActionVehicleTeleport`, …)
share this dispatch table but are defined in their own GC files; they get their
own `docs/<file>/` entries when those modules are written up.
