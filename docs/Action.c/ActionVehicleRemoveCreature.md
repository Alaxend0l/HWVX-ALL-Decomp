# `ActionVehicleRemoveCreature`

## Signature

* **GC** `s32 ActionVehicleRemoveCreature(void)` — `0x800058B4`, size `0x8`
  (`HWVX-GC-Decomp/src/Action.c:51`).
* **PS2** `s32 ActionVehicleRemoveCreature(void)` — `0x0024F460`, size `0x8`
  (m2c draft `progress/best/ActionVehicleRemoveCreature.c`).
* **PC** not yet matched.

## Execution

```c
s32 ActionVehicleRemoveCreature(void) {
    return 1;
}
```

**No-op stub** — group member (see `ActionSpeakerChangeMode.md`). The
counterparts that *do* work are `ActionVehicleRemoveItem` (GC
`0x8000659C`/0xCC, PS2 `0x0024E640`/0xFC) and
`ActionCreatureChangeMode` (GC `0x80006AE0`/0x120, PS2 `0x0024E090`/0x104);
this removal hook itself is inert in both console builds.

## Version differences

* 0x8 → 0x8, identical stub shape. Address ordering in the PS2 image is the
  **reverse** of GC for this stub block (PS2 runs descending
  `0x0024F490 Speaker → … → 0x0024F450 ScriptLight`; GC runs ascending
  `0x8000589C → 0x800058BC`) — link order, no behavioural meaning, but worth
  knowing when splitting `cod` into subsegments.
