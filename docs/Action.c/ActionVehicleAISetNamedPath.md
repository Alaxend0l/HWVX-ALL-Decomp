# `ActionVehicleAISetNamedPath`

## Signature

* **GC** `s32 ActionVehicleAISetNamedPath(void)` — `0x800058A4`, size `0x8`
  (`HWVX-GC-Decomp/src/Action.c:43`).
* **PS2** `s32 ActionVehicleAISetNamedPath(void)` — `0x0024F480`, size `0x8`
  (m2c draft `progress/best/ActionVehicleAISetNamedPath.c`).
* **PC** not yet matched.

## Execution

```c
s32 ActionVehicleAISetNamedPath(void) {
    return 1;
}
```

**No-op stub** (see `ActionSpeakerChangeMode.md` for the group explanation):
the "assign this AI vehicle to a named nav path" hook is present in the
dispatch table but the shipped body ignores the payload entirely.

## Version differences

* Identical shape and size (0x8 → 0x8) on GC and PS2 — same stub group
  (`ActionSpeakerChangeMode` · this · `ActionSetPlayerVehicle`, addresses
  `0x8000589C`–`0x800058BC` GC / `0x0024F450`–`0x0024F490` PS2).
* The callee machinery it *would* call (`NavPointChangeMode`-adjacent code,
  `ActionNavPointChangeMode` at GC `0x80006…`) does exist in the build, so the
  stub is deliberate game-side dead code, not a link-time casualty.
