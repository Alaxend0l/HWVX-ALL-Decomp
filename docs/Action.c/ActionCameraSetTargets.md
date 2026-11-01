# `ActionCameraSetTargets`

## Signature

* **GC** `s32 ActionCameraSetTargets(void)` — `0x800057E0`, size `0x24`.
* **PS2** `s32 ActionCameraSetTargets(void)` — `0x24F510`, size `0x20`
  (m2c draft `HWVX-PS2-Decomp/progress/best/ActionCameraSetTargets.c`).
* **PC** not yet matched.

## Arguments

None. As an action handler it *receives* `data` in the first argument register
from `Action_Heartbeat` but ignores it.

## Execution

One-shot bridge from the action registry into the camera system:

```c
s32 ActionCameraSetTargets(void) {
    CameraManager_SetCameraGlobalMode();
    return 1;
}
```

Returns `1` — the heartbeat's return value is unused, but convention in this
family is "handled".

## Version differences

* Byte-identical intent, 0x24 vs 0x20: PPC's call+li sequence is one word
  longer here than MIPS's `jal` in the delay slot. No semantic difference.
* The callee `CameraManager_SetCameraGlobalMode` is itself shared-name
  (in both symbol tables), so the camera doc will anchor the same behaviour.
