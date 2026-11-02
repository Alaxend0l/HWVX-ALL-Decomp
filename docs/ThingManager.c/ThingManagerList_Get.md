# `ThingManagerList_Get`

Membership test: walks a thing list and returns `1` if `target` is itself a
list node (pointer identity, no field comparison), `0` otherwise.

## Signature

* **GC** `s32 ThingManagerList_Get(void *list, void *target)` —
  `0x80054010`, size `0x2C` (`HWVX-GC-Decomp/src/ThingManager.c:128`).
* **PS2** — `0x002B1850`, size `0x34`
  (`asm/nonmatchings/cod/ThingManagerList_Get.s`).
* **PC** — `0x0044A870`, extent `0x30` (body ends `ret` @`0x0044A891`), in
  unit `text_0044A0D0`.

## Arguments

`list` = pointer to a list-head slot; `target` = candidate thing pointer
(itself, not a field value — contrast `ThingManagerList_GetByType`).

## Execution

```c
s32 ThingManagerList_Get(void *list, void *target) {
    void *thing;

    for (thing = *(void **)list; thing != NULL; thing = M2C_FIELD(thing, void **, 0xA4)) {
        if (target == thing) {
            return 1;
        }
    }
    return 0;
}
```

## PC match

```
0x0044a870: mov eax, [esp + 4]
0x0044a874: mov eax, [eax]            ; thing = *list
0x0044a876: test eax, eax
0x0044a878: je  0x44a88c              ; empty -> return 0
0x0044a87a: mov ecx, [esp + 8]        ; target (hoisted out of loop)
0x0044a87e: cmp ecx, eax
0x0044a880: je  0x44a88f              ; found -> return 1
0x0044a882: mov eax, [eax + 0xa4]     ; thing = next
0x0044a888: test eax, eax
0x0044a88a: jne 0x44a87e
0x0044a88c: xor al, al
0x0044a88e: ret
0x0044a88f: mov al, 1
0x0044a891: ret
```

## Version differences

* **Return-value width.** PS2 writes the full register: `addiu $2,$0,0x1`
  @`0x002B1868` sits in the delay slot of the epilogue jump `b .L002B187C`,
  and the zero path falls through `daddu $2,$0,$0` @`0x002B1878` into the
  shared `jr $31`. PC MSVC only writes the low byte — `xor al,al`
  @`0x0044A88C` and `mov al,1` @`0x0044A88F` leave the upper 24 bits of
  `eax` as leftovers from the `mov eax,[eax]` walk. Any PC caller must
  therefore treat the result as a byte/nonzero test, never compare it
  against a full dword. GC has the same source (`return 1`/`return 0`), so
  this is pure codegen: MSVC proving nobody reads bits 8–31.
* **Found-path shape.** PS2 exits through an unconditional `b` whose delay
  slot carries the `1` (@`0x002B1864`/`0x002B1868`); PC duplicates the `ret`
  instruction per exit (@`0x0044A88E` / `0x0044A891`); GC keeps one epilogue.
* All three compare the node pointer against `target` directly with no
  memory access on the target side; PC hoists the `target` argument load
  above the loop (@`0x0044A87A`), PS2 already has it in `$5`.
