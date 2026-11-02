# `ThingManager_DARYL_CreateThing`

Allocates a thing of pool `type` from `freeList[type]`, links it at the head
of the global active list, then runs its class init (`vtable[0]`). Returns
`NULL` when the pool is exhausted — **callers must NULL-check**, no platform
bounds-checks or asserts.

## Signature

* **GC** `void *ThingManager_DARYL_CreateThing(s32 type, void *params)` —
  `0x80057C18`, size `0xC8` (`HWVX-GC-Decomp/src/ThingManager.c:1402`).
* **PS2** — `0x002AD5E0`, size `0xB0`
  (`asm/nonmatchings/cod/ThingManager_DARYL_CreateThing.s`).
* **PC** — `0x0044D0B0`, extent `0x40` (unit `text_0044D050` — separate
  MSVC unit from the list cluster, see `_README.md`).

## Arguments

`type` = pool index 0–17 (`_README.md` table); `params` = pointer passed
verbatim to the class init — its meaning is per-vtable (level item record,
spawn position block, …). `params` may be `NULL` for classes whose init
ignores it.

## Execution

GC source (with the two inlined list ops annotated):

```c
void *ThingManager_DARYL_CreateThing(s32 type, void *params) {
    void *thing = g_thingManager_DARYL.freeList[type];

    if (thing == NULL) {
        thing = NULL;                     /* redundant, line 1405-1406 */
    } else {                              /* --- inlined Pop(freeList[type]) */
        g_thingManager_DARYL.freeList[type] = M2C_FIELD(thing, void **, 0xA4);
        if (g_thingManager_DARYL.freeList[type] != NULL) {   /* re-read! */
            M2C_FIELD(g_thingManager_DARYL.freeList[type], void **, 0xA0) = NULL;
        }
        M2C_FIELD(thing, void **, 0xA4) = NULL;
    }
    if (thing != NULL) {                  /* --- inlined Append(&activeList) */
        M2C_FIELD(thing, void **, 0xA4) = g_thingManager_DARYL.activeList;
        M2C_FIELD(thing, void **, 0xA0) = NULL;
        if (M2C_FIELD(thing, void **, 0xA4) != NULL) {
            M2C_FIELD(M2C_FIELD(thing, void **, 0xA4), void **, 0xA0) = thing;
        }
        g_thingManager_DARYL.activeList = thing;
        ((void (*)(void *, void *))(M2C_FIELD(thing, void ***, 0x8C))[0])(thing, params);
        return thing;
    }
    return NULL;
}
```

Order matters: the thing is **on the active list before `Init` runs**, so an
init that walks the active list (or `ThingManagerList_GetByType`) already
sees it. `Init` is free to fail/`Thing_Release` itself and the sweep removes
it next frame.

## PC match

```
0x0044d0b0: mov eax, [esp + 4]            ; type
0x0044d0b4: lea ecx, [eax*4 + 0x53a9d4]   ; &freeList[type]
0x0044d0bb: push esi
0x0044d0bc: push ecx
0x0044d0bd: call 0x44a810                 ; ThingManagerList_Pop
0x0044d0c2: mov esi, eax                  ; thing
0x0044d0c4: add esp, 4
0x0044d0c7: test esi, esi
0x0044d0c9: je  0x44d0eb                  ; exhausted -> NULL
0x0044d0cb: push esi
0x0044d0cc: push 0x53a9d0                 ; &activeList
0x0044d0d1: call 0x44a790                 ; ThingManagerList_Append
0x0044d0d6: mov eax, [esp + 0x14]         ; params (push-shifted)
0x0044d0da: mov edx, [esi + 0x8c]         ; vtable
0x0044d0e0: push eax
0x0044d0e1: push esi
0x0044d0e2: call dword ptr [edx]          ; vtable[0](thing, params)
0x0044d0e4: add esp, 0x10                 ; 4 pushes in one cleanup
0x0044d0e7: mov eax, esi
0x0044d0e9: pop esi
0x0044d0ea: ret
0x0044d0eb: xor eax, eax
0x0044d0ed: pop esi
0x0044d0ee: ret
```

Globals: `activeList` `0x0053A9D0`, `freeList[]` `0x0053A9D4` — together with
GC's struct offsets (`+0x7AB70`/`+0x7AB74`) these pin the PC manager base to
`0x004BFE60` (no symbol; see `_README.md`). This function is the module's
best PC fingerprint: two list-helper calls + those two globals is a unique
pattern in the binary.

## Version differences

* **Factoring**: PC calls `ThingManagerList_Pop` (`0x0044D0BD`) +
  `ThingManagerList_Append` (`0x0044D0D1`); GC/PS2 inline both blocks (see
  `ThingManagerList_Pop.md` for the identical inline bodies).
* **PS2 narrows `type` to a byte**: `andi $2,$4,0xFF` @`0x002AD5E0` before
  `sll $3,$2,2` @`0x002AD5E8`; GC/PC index with the full 32-bit value (PC
  `lea ecx,[eax*4+0x53a9d4]` @`0x0044D0B4`). PS2 callers effectively pass
  `u8` type.
* **PS2 shows the redundant `thing = NULL` (GC line 1405–1406) as real
  code**: `b .L002AD630` / delay `daddu $16,$0,$0` @`0x002AD60C`/`0x002AD610`
  zeroes the callee-saved `thing` register on the empty-pool path; PC's
  equivalent is just `xor eax,eax` @`0x0044D0EB`.
* **PS2 re-read of `freeList[type]`** (mirrors GC line 1409):
  `lw $2,0x0($3)` @`0x002AD61C` reloads the value stored one instruction
  earlier. PC hides this inside `Pop`, which instead tests the cached
  `head->next`.
* **PS2 addresses `activeList` two different ways inside one function**: the
  head *load* computes manager base + `0x7AB70` explicitly (`lui %hi(D_66A030)`
  + `ori $3,0x7AB70` + `addu` @`0x002AD63C`–`0x002AD648`, confirming the PS2
  manager base `0x0066A030`), while the head *store* uses the absolute global
  `D_006E4BA0` (hi/lo split across the `beqz` delay slot `lui`
  @`0x002AD660` + `sw` @`0x002AD668`). Both names are `base+0x7AB70` —
  inconsistent linker relaxation, not two variables.
* **Init call ABI**: PS2 `jalr $2` @`0x002AD674` with `daddu $4,$16,$0` in
  the delay slot (`$4` = this; `$5` = `params` survives untouched from the
  caller); PC pushes `(this, params)` in cdecl order @`0x0044D0E0`–`0x0044D0E1`;
  GC ABI is r3/r4. All three call `[vtable + 0x00]`.
* Save-area style: PS2 `sq $16` + `sd $31` frame (`0x20` bytes,
  `lq $16`/`ld $31` restore @`0x002AD680`–`0x002AD688`); PC only spills `esi`.
