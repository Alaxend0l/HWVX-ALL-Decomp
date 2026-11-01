# `ActionInterfacePlay`

## Signature

* **GC** `s32 ActionInterfacePlay(s32 *arg0)` — `0x8003F120`, size `0x30`
  (defined in `HWVX-GC-Decomp/src/InterfaceManager.c:5650` — the GC split
  attaches it to the *file containing that link region*, not `Action.c`).
* **PS2** `s32 ActionInterfacePlay(s32 *arg0)` — `0x00286A50`, size `0x28`
  (m2c draft `progress/best/ActionInterfacePlay.c`).
* **PC** not yet matched — PC has no symbols; match this against GC's body
  and rename the `_func_XXXXXXXX` entry when confirmed.

## Arguments

`arg0` — record's `data`; `*arg0` = interface (HUD element / message
resource) id.

## Execution

```c
s32 ActionInterfacePlay(s32 *arg0) {
    InterfaceManager_Create(*arg0, 0, 0);   /* id, player=0, flags=0 */
    return 1;
}
```

Spawns one interface element through the interface manager; the two zero args
are the fixed player/flags defaults for scripted ("played") elements.

## Version differences

* 0x30 vs 0x28, same call shape: PS2 loads `*arg0` into `$4`, zeroes
  `$5`/`$6` (one via the `daddu` delay slot), `jal InterfaceManager_Create`;
  GC spills args to its frame. No semantic difference.
* Link placement mirrors between builds again (far from the `Action*` block:
  GC `0x8003F120`, PS2 `0x00286A50`) — when splitting PS2 `cod` toward GC
  files, this one belongs beside the `InterfaceManager_*` functions, not the
  Action block.
