# `Action_ReleaseAllActions`

## Signature

* **GC** `void Action_ReleaseAllActions(void)` — `0x80005A6C`, size `0x30`
  (`HWVX-GC-Decomp/src/Action.c:104`).
* **PS2** `void Action_ReleaseAllActions(void)` — `0x24F300`, size `0x14`
  (m2c draft `HWVX-PS2-Decomp/progress/best/Action_ReleaseAllActions.c`).

## Arguments

None.

## Execution

Dumpster-clears the registry — used at level teardown / restart so queued
actions cannot leak across levels:

```c
void Action_ReleaseAllActions(void) {
    memset(m_Actions_800DBC80, 0, 0x140);   /* 80 pointer slots */
}
```

* **GC** calls the SDK `memset` with an explicit byte count (`0x140`) — hence
  the 0x30 bytes.
* **PS2** calls `memset(&D_518D50, 0)` (draft shows the count argument
  unresolved as an extern `memset(M2C_UNK*, M2C_UNK)`; the 0x14-byte body is
  just load-address + 2 args + tail call). The cleared region is the same
  0x140-byte table, confirmed by `Action_Heartbeat`/`Action_AddAction` which
  bound that table at 0x50 slots × 4.

## Version differences

* **Same effect, inverted size delta** (0x30 → 0x14): GC's call sequence sets
  up three arguments plus a branch-back, PS2 MIPS tail-calls `memset` in the
  delay slot. Nothing to learn about the game logic from the size difference.
* GC writes the target as a *named* file-static (`m_Actions_800DBC80`); PS2
  only ever has `D_518D50` — name the PS2-side table after the GC symbol in
  future split code.
