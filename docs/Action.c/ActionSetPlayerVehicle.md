# `ActionSetPlayerVehicle`

## Signature

* **GC** `s32 ActionSetPlayerVehicle(void)` — `0x800058AC`, size `0x8`
  (`HWVX-GC-Decomp/src/Action.c:47`).
* **PS2** `s32 ActionSetPlayerVehicle(void)` — `0x0024F470`, size `0x8`
  (m2c draft `progress/best/ActionSetPlayerVehicle.c`).
* **PC** not yet matched.

## Execution

```c
s32 ActionSetPlayerVehicle(void) {
    return 1;
}
```

**No-op stub** — same stub group as `ActionSpeakerChangeMode` /
`ActionVehicleAISetNamedPath`: the "give the player this scripted vehicle" hook
is dispatched and cleared but does nothing in the shipped build (vehicle
swapping actually goes through `ActionVehicleChangeMode`, which is a real
0x3bc/0x45c-byte handler).

## Version differences

* 0x8 → 0x8, byte-shape identical modulo ISA. One of the few exact size
  matches between GC and PS2.
