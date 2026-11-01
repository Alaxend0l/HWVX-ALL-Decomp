# `ActionSetPlayerOrder`

## Signature

* **GC** `s32 ActionSetPlayerOrder(void *action)` — `0x80005874`, size `0x28`
  (`HWVX-GC-Decomp/src/Action.c:34`).
* **PS2** `s32 ActionSetPlayerOrder(…)` — `0x0024F4A0`, size `0x20`
  (m2c draft `progress/best/ActionSetPlayerOrder.c`, asm
  `asm/nonmatchings/cod/ActionSetPlayerOrder.s`).
* **PC** not yet matched.

## Arguments

`action` — the record's `data` array. The rules call is
`GameRules_SetPlayerOrder(data, 4)`: the mode constant **4** = "scripted player
order" (as opposed to race-start/random assignment), and `data` carries the
order payload the level script queued.

## Execution

```c
s32 ActionSetPlayerOrder(void *action) {
    GameRules_SetPlayerOrder(action, 4);
    return 1;
}
```

## Version differences

* **Not a real difference — a draft artifact.** GC's source passes two args
  `(action, 4)`; PS2's m2c draft reads `GameRules_SetPlayerOrder(4)`, and the
  reason is visible in the asm: the body sets **only**
  `addiu $5, $0, 0x4` before `jal GameRules_SetPlayerOrder` — `$4` (first
  argument) is never written, so it is the leftover `data` pointer `Action_Heartbeat`
  parked there. PS2 makes the **same two-argument call**; a hand-written PS2
  decomp must keep `action` in the signature, not "drop the argument".
* Sizes 0x28 vs 0x20: PPC spills/loads the first arg through the frame (plus
  `blr`-pair bookkeeping); MIPS passes `$4` through untouched. Callee sizes
  differ slightly too (`GameRules_SetPlayerOrder` 0xA4 GC → 0xAC PS2).
