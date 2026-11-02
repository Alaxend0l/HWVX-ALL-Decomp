# `ThingManagerList_GetByType`

Walks a list and returns the first thing whose `type` field (`0xA8`) equals
the requested type; `NULL` when none matches.

## Signature

* **GC** `void *ThingManagerList_GetByType(void *list, s32 type)` —
  `0x80053FE8`, size `0x28` (`HWVX-GC-Decomp/src/ThingManager.c:117`).
* **PS2** — `0x002B1890`, size `0x3C`
  (`asm/nonmatchings/cod/ThingManagerList_GetByType.s`).
* **PC** — `0x0044A840`, extent `0x30` (body ends `ret` @`0x0044A862`), in
  unit `text_0044A0D0`.

## Arguments

`list` = pointer to a list-head slot; `type` = free-list index (see
`_README.md` for the 0–17 table).

## Execution

```c
void *ThingManagerList_GetByType(void *list, s32 type) {
    void *thing;

    for (thing = *(void **)list; thing != NULL; thing = M2C_FIELD(thing, void **, 0xA4)) {
        if (M2C_FIELD(thing, s32 *, 0xA8) == type) {
            return thing;
        }
    }
    return NULL;
}
```

## PC match

```
0x0044a840: mov eax, [esp + 4]
0x0044a844: mov eax, [eax]            ; thing = *list
0x0044a846: test eax, eax
0x0044a848: je  0x44a860              ; empty -> xor eax,eax
0x0044a84a: mov ecx, [esp + 8]        ; type (hoisted out of loop)
0x0044a84e: cmp dword ptr [eax + 0xa8], ecx
0x0044a854: je  0x44a862              ; found -> ret (eax = thing)
0x0044a856: mov eax, [eax + 0xa4]     ; thing = next
0x0044a85c: test eax, eax
0x0044a85e: jne 0x44a84e
0x0044a860: xor eax, eax
0x0044a862: ret
```

## Version differences

* **PS2 compares the type as a BYTE — GC and PC as a full dword.** PS2 masks
  the argument (`andi $3,$5,0xFF` @`0x002B1898`, executed in the entry test's
  delay slot so even the empty-list path pays for it) and loads the field
  with `lbu $4,0xA8($2)` @`0x002B189C`; GC/PC compare 32-bit (`cmp dword ptr
  [eax+0xa8],ecx` @`0x0044A84E`). Consequence: a caller that passes
  `type > 255`, or a thing whose `0xA8` upper bytes are ever dirty, matches
  on GC/PC but not on PS2. This byte-narrowing recurs across the PS2 Thing
  cluster (`CreateThing` `andi $2,$4,0xFF` @`0x002AD5E0`, `CreateSlave`
  @`0x002AD544`) — treat PS2 `type` as `u8` semantics when porting.
* **NULL tail merge**: PS2 falls into `daddu $2,$0,$0` @`0x002B18C0` on
  exhaustion while the found path branches around it (`b .L002B18C4`
  @`0x002B18A8`); PC merges the same way with `xor eax,eax` @`0x0044A860`
  falling into the shared `ret` @`0x0044A862`. GC is the plain source loop.
* PC hoists the `type` argument load above the loop (@`0x0044A84A`); PS2's
  argument is already in `$5`; GC's m2c draft shows the constant folded per
  iteration only because the optimiser reused a register.
