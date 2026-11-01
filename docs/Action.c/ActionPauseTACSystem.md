# `ActionPauseTACSystem`

## Signature

* **GC** `s32 ActionPauseTACSystem(s32 *action)` — `0x80006420`, size `0x78`
  (`HWVX-GC-Decomp/src/Action.c:510`; declared callee
  `Trigger_PauseTACSystem(s8)` at src line 508).
* **PS2** `s32 ActionPauseTACSystem(s32 *arg0)` — `0x0024E860`, size `0x80`
  (m2c draft `progress/best/ActionPauseTACSystem.c`).
* **PC** not yet matched — match against GC (no PC debug symbols exist; the
  bodies are expected near-identical).

## Arguments

`action` = record's `data`; `action[0]` = mode code: **7** = pause,
**8** = unpause, **2** = toggle. Any other code falls through (no-op).

## Execution

```c
s32 ActionPauseTACSystem(s32 *action) {
    switch (action[0]) {
    case 7: Trigger_PauseTACSystem(1); break;
    case 8: Trigger_PauseTACSystem(0); break;
    case 2: Trigger_PauseTACSystem(!Trigger_IsTACSystemPaused()); break;
    }
    return 1;
}
```

TAC = the traffic/AI-controller system in `Trigger_*`; the action either
pauses, resumes, or flips it depending on the current state.

## Version differences

* **Same three cases, same constants, same fall-through-on-unknown** (0x78
  GC vs 0x80 PS2 — MIPS just needs a wider jump-table/branch pad; GC
  `0x78` is the tighter PPC case chain).
* The toggle arm's C spelling differs only by draft noise: GC draft
  `!Trigger_IsTACSystemPaused()`; PS2 m2c draft
  `Trigger_PauseTACSystem((s64)((Trigger_IsTACSystemPaused() == 0) << 0x38) >>
  0x38)` — m2c inventing sign-extension scaffolding around a byte-typed
  boolean. Write `(s8)!Trigger_IsTACSystemPaused()` on both sides; the
  `<<0x38 >>0x38` dance is not game code.
* Both share callee names `Trigger_PauseTACSystem` /
  `Trigger_IsTACSystemPaused`, so the `Trigger.c` doc anchors this once.
