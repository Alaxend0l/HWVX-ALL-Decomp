# `ActionRoundComplete`

## Signature

* **GC** `s32 ActionRoundComplete(s32 *arg0)` — `0x80005804`, size `0x28`.
* **PS2** `s32 ActionRoundComplete(s32 *arg0)` — `0x24F4F0`, size `0x20`
  (m2c draft `progress/best/ActionRoundComplete.c`, already a clean decomp).

## Arguments

`arg0` — the action record's `data` array (first slot = round-end reason / mode
passed to the rules layer as an `s32`).

## Execution

```c
s32 ActionRoundComplete(s32 *arg0) {
    GameRules_ForceRoundEnd(*arg0);   /* dereference: pass data[0] by value */
    return 1;
}
```

The action's whole job is to force the round-end path in `GameRules` with the
reason code carried in the record.

## Version differences

* **No logic difference** (0x28 vs 0x20 = ISA call encoding; PS2 loads
  `0($4)` into `$5` pre-`jal`, GC `lwz`+`stw` prologue for the arg spill).
* Both sides call the same name (`GameRules_ForceRoundEnd` exists in GC and
  PS2 symbol tables), so the rules layer can be documented once.
