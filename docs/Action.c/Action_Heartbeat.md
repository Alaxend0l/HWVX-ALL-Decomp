# `Action_Heartbeat`

## Signature

* **GC** `void Action_Heartbeat(void)` — `0x800059A8`, size `0xc4`.
* **PS2** `void Action_Heartbeat(void)` — `0x24F320`, size `0xd0`
  (`asm/nonmatchings/cod/Action_Heartbeat.s`).
* **PC** not yet matched.

## Arguments

None — the per-frame delta time is read from a global.

## Execution

This is the per-frame tick of the action registry; the game calls it once per
frame from the main loop.

**GC** (`HWVX-GC-Decomp/src/Action.c:74`-`94`):

```c
for (i = 0; i < 80; i++) {
    Action *action = m_Actions_800DBC80[i];
    if (action != NULL && action->delay <= 0) {
        pActionFuncArray_800CF500[action->type](action->data);
        if (m_Actions_800DBC80[i]->type != 0x14 ||
            ((void **)m_Actions_800DBC80[i]->data)[1] == NULL)
            m_Actions_800DBC80[i] = NULL;
    } else if (action != NULL) {
        action->delay -= g_iDeltaTimeMS;
    }
}
```

* Slot empty (`NULL`) → skip.
* `delay <= 0` → **fire** the record: dispatch through the function-pointer
  table indexed by the record's `type` field, passing the record's `data`
  array. Then the slot is *cleared* unless the record is the "repeating" type
  `0x14` with a non-NULL `data[1]` (that family keeps itself registered; its
  handler re-arms the delay).
* `delay > 0` → count the frame down by `g_iDeltaTimeMS`.

**PS2** (`0x24F320`, hand-decoded from the asm): identical algorithm, field for
field —

| step | PS2 instruction |
| --- | --- |
| table walk | `D_518D50 + i*4`, bound `0x50` (80 slots, stride 4 — same shape as GC) |
| empty-slot test | `lw $5, 0($18)` ; `beqz` |
| due test | `lw $3, 0x14($5)` ; `bgtz` (delay at **0x14**, GC's record offset) |
| dispatch | `lw $3, 0($5)` → `type * 4` → `D_002C4F90[type]` → `jalr`, arg from `0x4($5)` (delay slot) = `data` |
| re-arm guard | `type != 0x14` → clear slot; else `lw 0x4(data_p)` → `data[1]`; `data[1] != 0` → keep slot |
| countdown | `lw $4, 0x14($5)` − **gp-relative load** (`%gp_rel`, unresolved, ≈`g_iDeltaTimeMS`) → `sw` back at `0x14` |

## Version differences

* **Same semantics, same record field offsets** (type `0x0`, data `0x4`,
  delay `0x14`); the delta `0xc4 → 0xd0` is codegen: PS2 pays a 0x40-byte
  stack frame with `sd`/`sq` callee saves, and the frame-delta global is
  `%gp_rel`-addressed (`0x002D00B0` in the asm, which splat could not
  symbolize — its `gp` guess was off), while GC's is a plain absolute load of
  `g_iDeltaTimeMS`.
* The repeating-type constant `0x14` and the "re-arming" rule are
  **verified identical** across the two platforms, so action-type IDs can be
  documented once (GC) and reused on PS2.
* No PC reference yet; treat the PC `functions.json` scan for a two-level
  pointer-table walk as the way to match it once units are renamed.
