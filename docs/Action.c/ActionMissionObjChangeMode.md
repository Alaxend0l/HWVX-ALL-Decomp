# `ActionMissionObjChangeMode`

## Signature

* **GC** `s32 ActionMissionObjChangeMode(u32 *action)` — `0x80005FF8`, size `0x38`
  (`HWVX-GC-Decomp/src/Action.c:296`).
* **PS2** `s32 ActionMissionObjChangeMode(s32 *arg0)` — `0x0024ED80`, size `0x40`
  (`asm/nonmatchings/cod/ActionMissionObjChangeMode.s`; no m2c draft).
* **PC** not yet matched (no PC symbols; match against GC, see
  `docs/_meta/version-differences.md`).

## Arguments

`action` = record's `data`: `action[0]` = mode byte, `action[1]` = list
length, `action[2 + i]` = pointer to a mission-object record; each
`*(action[i+2] + 4)` is a further pointer and the mode is written as **one
byte** at that record's offset `0x1`.

## Execution

```c
s32 ActionMissionObjChangeMode(u32 *action) {
    u32 i;

    for (i = 0; i < action[1]; i++) {
        M2C_FIELD(M2C_FIELD(action[i + 2], u8 **, 4), u8 *, 1) = action[0];
    }
    return 1;
}
```

Two-stage list setter: the record holds an array of pointers; each entry is
followed once (`+4`) and the mode byte stored into the pointed-to object.
(`ActionNavPointChangeMode` is the same handler family with **one** level of
indirection — see the sibling page.)

## Version differences

* **0x38 GC vs 0x40 PS2** — PS2 pads the loop-skip `b`/`nop` pair and a
  `nop` at the bottom-test entry; behaviour is word-identical.
* PS2's loop shape: top `b .L0024EDA4` skip → bottom test (`lw $2,4($4)`
  reloads `action[1]` **every iteration**, `sltu`, `bnez` with `addu $2,$4,$6`
  in the delay slot) → body. Keep `i < action[1]` re-reading the field
  (hoisting a cached count changes the emitted shape and the semantics if the
  record ever aliases itself).
* The iteration index is kept **twice**: `$5` counts iterations
  (`addiu $5,$5,1`) and `$6` accumulates the **byte** offset
  (`addiu $6,$6,4`); the pointer is then materialized as
  `addu $2,$4,$6` → `lw $2,8($2)`. The `8` folds "start at `action[2]`" into
  the load — with i=0, `arg+0x8` is exactly `action[2]`, matching GC's
  `action[i + 2]`.
* Mode fetch is **byte-wide**: `lbu $3,0($4)` (not `lw`) and the store is
  `sb $3,0x1($2)` — GC's `= action[0]` is a u32 expression stored into a
  byte slot; a decomp typed `= (u8)action[0]` or a u32 store would both alter
  codegen.
* Return `1` is the `jr $31` delay-slot `addiu $2,$0,0x1` at
  `0x24EDBC` — no branch around it.
