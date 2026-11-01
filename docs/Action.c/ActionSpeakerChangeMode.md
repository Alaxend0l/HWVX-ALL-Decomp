# `ActionSpeakerChangeMode`

## Signature

* **GC** `s32 ActionSpeakerChangeMode(void)` — `0x8000589C`, size `0x8`
  (`HWVX-GC-Decomp/src/Action.c:39`).
* **PS2** `s32 ActionSpeakerChangeMode(void)` — `0x0024F490`, size `0x8`
  (m2c draft `progress/best/ActionSpeakerChangeMode.c`).
* **PC** not yet matched.

## Execution

```c
s32 ActionSpeakerChangeMode(void) {
    return 1;
}
```

A **do-nothing handler**: the type exists in the action vocabulary (level
scripts can still queue it, and `Action_Heartbeat` still dispatches and
clears it), but the shipped code does nothing with the payload. Whatever
speaker-mode plumbing existed was either removed upstream or never enabled in
this build.

## Version differences

* **Byte-for-byte the same shape, and the same size (0x8 = 0x8)** — one of the
  rare exact size matches across ISAs (`bl`/`li r3,1` on PPC vs
  `jr $31`/`addiu $2,1` on MIPS; the two trailing `nop` words in the PS2 asm
  are alignment padding, outside the symbol's size).
* Shares this stub group with
  `ActionVehicleAISetNamedPath`, `ActionSetPlayerVehicle`,
  `ActionVehicleRemoveCreature`, `ActionScriptLightChangeMode` (adjacent
  addresses, adjacent slots in the dispatch table). Treat the five as one
  documented behaviour with five names.
