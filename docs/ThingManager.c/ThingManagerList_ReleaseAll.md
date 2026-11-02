# `ThingManagerList_ReleaseAll`

Walks a thing list and calls each node's vtable slot `[1]` (`Release(this)`).
Releases only — nodes are **not** unlinked; they stay on `list` (usually now
flagged `0x200000` by their `Release`) for a later
`ThingManagerList_RemoveReleasedThings` sweep.

## Signature

* **GC** `void ThingManagerList_ReleaseAll(void **list)` — `0x8005446C`, size
  `0x4C` (`HWVX-GC-Decomp/src/ThingManager.c:202`; forward-prototyped at
  `:183` above `ThingManagerList_ReleaseAndRemoveAll`).
* **PS2** — `0x002B15C0`, size `0x48`
  (`asm/nonmatchings/cod/ThingManagerList_ReleaseAll.s`).
* **PC** — `0x0044A990`, extent `0x30` (body ends `ret` @`0x0044A9B5`), in
  unit `text_0044A0D0`.

## Arguments

`list` = pointer to a list-head slot. No return value.

## Execution

```c
void ThingManagerList_ReleaseAll(void **list) {
    void *thing;
    void *next;

    thing = *list;
    while (thing != NULL) {
        next = M2C_FIELD(thing, void **, 0xA4);
        M2C_FIELD(M2C_FIELD(thing, void **, 0x8C), void (**)(void *), 4)(thing);
        thing = next;
    }
}
```

## PC match

```
0x0044a990: mov eax, [esp + 4]          ; list
0x0044a994: mov eax, [eax]              ; thing = *list
0x0044a996: test eax, eax
0x0044a998: je  0x44a9b5                ; empty -> ret, skipping push/pop esi
0x0044a99a: push esi
0x0044a99b: mov ecx, [eax + 0x8c]       ; vtable
0x0044a9a1: mov esi, [eax + 0xa4]       ; next — captured BEFORE the call
0x0044a9a7: push eax                    ; this
0x0044a9a8: call dword ptr [ecx + 4]    ; vtable[1] Release(thing)
0x0044a9ab: add esp, 4                  ; per-call cleanup (loop has one call)
0x0044a9ae: test esi, esi
0x0044a9b0: mov eax, esi
0x0044a9b2: jne 0x44a99b                ; re-enters AFTER push esi
0x0044a9b4: pop esi
0x0044a9b5: ret
```

## Version differences

* **`next` before the virtual call** on all three — mandatory, since
  `Release` may mutate `next`: GC source statement order; PS2
  `lw $16,0xA4($4)` in the `jalr $2` delay slot @`0x002B15E4`; PC
  `mov esi,[eax+0xa4]` @`0x0044A9A1` before `push eax`/`call [ecx+4]`
  @`0x0044A9A7`/`0x0044A9A8`.
* **PS2 body**: frame `addiu $29,$29,-0x20` @`0x002B15C0` saving `$31` (`sd`)
  and `$16` (`sq`); loop `lw $2,0x8C($4)`/`lw $2,4($2)`/`jalr $2`
  @`0x002B15D8`–`0x002B15E0`; three *wasted* `nop` delay slots (entry guard
  @`0x002B15D4`, loop back-edge `bnez` @`0x002B15F0`, guard landing
  @`0x002B15F4`); epilogue `jr $31` with `addiu $29,$29,0x20` in its delay
  slot @`0x002B1604`. That delay-slot instruction is the function's last
  real word — `0x002B1608`/`0x002B160C` are zero alignment padding (see
  `ThingManagerList_ReleaseAndRemoveAll.md`).
* **Byte-clone proof.** This exact loop is duplicated verbatim inside
  `ThingManagerList_RemoveReleasedThings` @`0x002B1654`–`0x002B166C` (only
  `next`'s register changes, `$16`→`$19`) — the PS2 compiler inlined
  `ReleaseAll` into the merged release-and-sweep function, which is how that
  function absorbed `ThingManagerList_ReleaseAndRemoveAll`'s job.
* **PC is leaf-shaped**: one guarded `push esi`/`pop esi`, no frame pointer,
  and per-call `add esp,4` (not batched — the loop body holds exactly one
  call). GC/PS2 save a link register (`$31` + callee-saved `$16` on PS2).
* **Callers** (PS2 `jal ThingManagerList_ReleaseAll`): `Thing_Release`
  @`0x002ABB0C`, `ThingManager_DARYL_Init` @`0x002AF5E0`/`0x002AF5F8`,
  `ThingManager_Invalidate` @`0x002AC3E8`,
  `ThingManager_MoveThings_DARYL_SUBTICK` @`0x002AF2E8`,
  `SharedSlaveHolder_Release` @`0x002B13B0`, `MotionState_Release`
  @`0x00255EF4` — i.e. every teardown path that must run destructors without
  touching list links.
