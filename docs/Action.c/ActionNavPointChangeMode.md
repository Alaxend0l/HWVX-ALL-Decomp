# `ActionNavPointChangeMode`

## Signature

* **GC** `s32 ActionNavPointChangeMode(u32 *action)` — `0x800063EC`, size `0x34`
  (`HWVX-GC-Decomp/src/Action.c:499`).
* **PS2** `s32 ActionNavPointChangeMode(s32 *arg0)` — `0x0024E8E0`, size `0x38`
  (`asm/nonmatchings/cod/ActionNavPointChangeMode.s`; no m2c draft — the two
  trailing nops at `0x24E918/0x24E91C` are link padding past `endlabel`, not
  part of the size).
* **PC** not yet matched (match against GC — no PC symbols exist).

## Arguments

`action` = record's `data`: `action[0]` = mode byte, `action[1]` = list
length, `action[2 + i]` = pointer to a nav-point record; the mode is written
as **one byte** directly at that record's offset `0x50`.

## Execution

```c
s32 ActionNavPointChangeMode(u32 *action) {
    u32 i;

    for (i = 0; i < action[1]; i++) {
        M2C_FIELD(action[i + 2], u8 *, 0x50) = action[0];
    }
    return 1;
}
```

Same skeleton as `ActionMissionObjChangeMode` but with **one** indirection
instead of two: the pointer array is followed once and the store lands at a
fixed `+0x50` slot inside the nav-point record (its mode/state field).

## Version differences

* **0x34 GC vs 0x38 PS2** — four bytes = PS2's `b`-to-test skip and the
  bottom-test `nop` (GC `0x34` is the tightest form of this handler family).
* PS2 loop skeleton is byte-for-byte the MissionObj shape: zero-init pair
  (`daddu $5,$0,$0` / `daddu $6,$0,$0` — iteration count and byte offset),
  top `b .L0024E900` skip, bottom test reloading `action[1]` via
  `lw $2,4($4)` + `sltu` + `bnez` with `addu $2,$4,$6` in the delay slot,
  body `addu`-based `lw $2,8($2)` (folds `action[2]` in), `lbu $3,0($4)`
  byte mode load, `sb $3,0x50($2)` store.
* Only the store displacement differs from MissionObj (`0x50` direct vs the
  extra `lw $2,4($2)` + `sb $3,0x1($2)` pair) — when matching PC, that pair
  is the single distinguishing feature between the two handlers.
* `return 1` in the `jr $31` delay slot (`addiu $2,$0,0x1` at `0x24E914`).
