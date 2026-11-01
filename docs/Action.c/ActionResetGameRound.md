# `ActionResetGameRound`

## Signature

* **GC** `s32 ActionResetGameRound(void *arg0)` — `0x8000582C`, size `0x48`.
* **PS2** `s32 ActionResetGameRound(void *arg0)` — `0x24F4C0`, size `0x2c`
  (m2c draft `progress/best/ActionResetGameRound.c`).

## Arguments

`arg0` — record's `data` array; three small ints at `0x0`, `0x4`, `0x8` carry
reset flags/modes (what the level script queued: e.g. checkpoint state, round
index, mode).

## Execution

```c
s32 ActionResetGameRound(void *arg0) {
    GameRules_ResetRound((s8)arg0[0], (s8)arg0[1], (s8)arg0[2]);
    return 1;
}
```

The three arguments are **byte-ranged**: GC loads them as 32-bit words and
truncates to `s8` (`(s8)(u8)M2C_FIELD(arg0, s32 *, 0)` in the decomp draft,
i.e. `lwz` + `extsb`); the rules API takes `s8` parameters.

## Version differences

* PS2 reads the same fields with **byte loads** (`lb` at 0x0/0x4/0x8 → draft
  writes `M2C_FIELD(arg0, s8 *, …)`), GC reads words and narrows — same values
  in either endianness (both big-endian), different instruction counts:
  0x48 vs 0x2c. Nothing to model game-side; just don't write a PS2 decomp with
  an `s32` load that then narrows — it costs the extra ops GC paid.
* The callee `GameRules_ResetRound(s8, s8, s8)` name/signature agrees on both
  symbol tables.
