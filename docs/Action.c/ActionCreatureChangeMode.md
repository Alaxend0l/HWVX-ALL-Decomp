# `ActionCreatureChangeMode`

## Signature

* **GC** `s32 ActionCreatureChangeMode(u32 *action)` — `0x80006AE0`, size `0x120`
  (`HWVX-GC-Decomp/src/Action.c:801`).
* **PS2** `s32 ActionCreatureChangeMode(s32 *action)` — `0x0024E090`, size `0x104`
  (`asm/nonmatchings/cod/ActionCreatureChangeMode.s`; no m2c draft).
* **PC** not yet matched (match against GC — no PC symbols exist).

## Arguments

`action` = record's `data`: `action[0]` = mode, `action[1]` = list length,
`action[2 + i]` = creature pointer. Each creature record: `+0x1C` = motion/
creature **handle** (0 = none), `+0x20` = creature mode word.

## Execution

```c
s32 ActionCreatureChangeMode(u32 *action) {
    u32 i, handle;
    u8 *creature;

    for (i = 0; i < action[1]; i++) {
        creature = (u8 *)action[i + 2];
        handle = M2C_FIELD(creature, u32 *, 0x1C);
        switch (action[0]) {
        case 0:                                  /* despawn */
        case 25:                                 /* alias of 0 */
            M2C_FIELD(creature, u32 *, 0x20) = 0;
            if (handle != 0) {
                M2C_FIELD(creature, u32 *, 0x1C) = 0;
                Creature_Release(handle);
            }
            break;
        case 1:                                  /* activate, keep handle */
            M2C_FIELD(creature, u32 *, 0x20) = 1;
            break;
        case 3:                                  /* send into death anim */
            M2C_FIELD(creature, u32 *, 0x20) = 0;
            if (handle != 0) {
                M2C_FIELD(creature, u32 *, 0x1C) = 0;
                Creature_MotionStateInitDeath((u8 *)handle);
            }
            break;
        case 26:                                 /* 1's store + 3's handle arm */
            M2C_FIELD(creature, u32 *, 0x20) = 1;
            if (handle != 0) {
                M2C_FIELD(creature, u32 *, 0x1C) = 0;
                Creature_MotionStateInitDeath((u8 *)handle);
            }
            break;
        }
    }
    return 1;
}
```

(GC draft lists `case 25`/`case 26` as separate labels with bodies identical
to `0`/`3`-with-`0x20=1`; written merged above for readability — keep the
five separate labels in the matched source if the diff tool insists.)

Mode summary: **0/25** = deactivate + release the creature handle; **1** =
activate only; **3** = deactivate + death-init; **26** = activate + death-init.
Unknown modes skip the switch but still count as a list entry.

## Version differences

* **PS2 is *smaller* than GC here (0x104 vs 0x120)** — unlike the setter
  pair, whose PS2 sides are padded; the PPC switch costs 28 more bytes than
  the MIPS compare cascade.
* **Cascade order is reversed and pre-staged**: PS2 tests **26, 25, 3, 1**
  with `beq` then **0** with `beqz`, then `b` default — GC source order is
  0/1/3/25/26. The next compare constant is staged in each branch's delay
  slot (`addiu $2,$0,0x19` after the 0x1A test, then 0x3, then 0x1), so
  `$2` is always the *next* constant at the moment of the test.
* **Codegen trap** (same family as the `$5` re-use noted in
  `ActionPrintDebugString.md`): the **case-1 arm stores `$2`** —
  `b .L0024E15C` / delay `sw $2,0x20($5)` at `0x24E110` — and `$2` is
  **1 only because it is the staged compare constant** from `addiu $2,$0,0x1`
  at `0x24E0DC`. The 26 arm restages it (`addiu $2,$0,0x1` at
  `0x24E148`) because it arrives with `$2` = 0x1A. A hand decomp that hoists
  or reorders constants silently changes what gets stored.
* **The handle load runs unconditionally every iteration**:
  `lw $4,0x1C($5)` sits in the delay slot of the *first* `beq`
  (`0x24E0C4`) — it executes even when the mode is unknown. Matches GC's
  hoisted `handle = ...` before the switch.
* **`sw $0,0x1C($5)` in the call delay slots** (`jal Creature_Release`
  `0x24E100`/`0x24E138`, `jal Creature_MotionStateInitDeath`
  `0x24E120`/`0x24E154`): the handle slot is zeroed *before* the callee
  starts — visible to the callee. Keep the clear-then-call order in the
  matched source.
* The `0x20 = 0` store of the handle arms is also a delay-slot store — the
  `beqz $4` delay slot at `0x24E0FC/0x24E11C/0x24E134` runs even when the
  handle is zero, so the mode word is cleared whether or not the call fires.
* Case-0 and case-25 arms are **separate code** (`0x24E0F8` and
  `0x24E130`, each with its own `jal Creature_Release`) — no compiler
  folding, and both arms stay 6 words apart; PS2 keeps its loop skeleton of
  the list-setter family (bottom test, `lw $2,4($18)` re-reads the count).
* Frame `0x40` with `sq $16/$17/$18` + `sd ra`; the return `addiu $2,$0,0x1`
  (`0x24E17C`) is not in a delay slot here — it sits after `ld ra`, before
  the three `lq` reloads and `jr $31`.
