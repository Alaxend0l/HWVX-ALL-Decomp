# `ActionScriptLightChangeMode`

## Signature

* **GC** `s32 ActionScriptLightChangeMode(void)` — `0x800058BC`, size `0x8`
  (`HWVX-GC-Decomp/src/Action.c:55`).
* **PS2** `s32 ActionScriptLightChangeMode(void)` — `0x0024F450`, size `0x8`
  (m2c draft `progress/best/ActionScriptLightChangeMode.c`).
* **PC** not yet matched.

## Execution

```c
s32 ActionScriptLightChangeMode(void) {
    return 1;
}
```

**No-op stub** — last member of the five-stub block
(`ActionSpeakerChangeMode`, `ActionVehicleAISetNamedPath`,
`ActionSetPlayerVehicle`, `ActionVehicleRemoveCreature`, this). Scripted light
changes are therefore inert in both console builds even though level data can
queue them; the lighting that does change goes through
`LightManager_*` on the per-frame path (the 0xef4/0x1658-byte `LightManager_LightThing`,
a renderer-side difference, not here).

## Version differences

* 0x8 → 0x8. PS2 places this stub **first** in the block (`0x0024F450`), GC
  **last** (`0x800058BC`) — same five names, mirrored link order.
