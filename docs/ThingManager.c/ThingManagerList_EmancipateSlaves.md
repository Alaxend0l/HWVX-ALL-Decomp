# `ThingManagerList_EmancipateSlaves`

Promotes slaves out of a thing's private child list into the world: for every
node whose `slavePlayer` byte (`0x83`) equals the sentinel `0xEF`, unlinks it
from `list`, transforms its world matrix in place
(`thing->mtx = thing->mtx * mtx`), prepends it onto the manager `activeList`,
and — for item things already placed in the world (`gridCell != 0`,
`vtable == g_itemVTable`) — re-adds them to the grid with type `3` and
clears the `8` (in-grid) flag. This is the *only* path onto `activeList` for
things created with `ThingManager_DARYL_CreateSlave` (which deliberately
skips the append).

## Signature

* **GC** `void ThingManagerList_EmancipateSlaves(void *mtx, void **list)` —
  `0x800544B8`, size `0x158` (`HWVX-GC-Decomp/src/ThingManager.c:214`).
* **PS2** — `0x002B1480`, size `0x13C`
  (`asm/nonmatchings/cod/ThingManagerList_EmancipateSlaves.s`).
* **PC** — `0x0044A9C0`, extent `0x90` (body ends `ret` @`0x0044AA4E`, next
  unit function starts `push edi` @`0x0044AA50`), in unit `text_0044A0D0`.

## Arguments

`mtx` = `Matrix4 *` world transform to merge into each slave's matrix (first
argument on all three platforms); `list` = pointer to a list-head slot
(typically `parentThing + 0x88`).

## Execution

```c
void ThingManagerList_EmancipateSlaves(void *mtx, void **list) {
    /* GC reconstruction carries a local SlaveThing typedef spelling out the
       ThingHeader offsets used here: mtx@0x10, flags@0x74, gridCell@0x80,
       slaveId@0x82, slavePlayer@0x83, vtable@0x8C, prev@0xA0, next@0xA4 */
    extern u8 g_itemVTable[];
    void MultiplyMatrix(void *a, void *b, Matrix4 *out);
    void Thing_AddToGrid(void *thing, s32 type);
    Matrix4 out;
    SlaveThing *thing;
    SlaveThing *next;

    thing = *list;
    while (thing != NULL) {
        next = thing->next;
        if (thing->slavePlayer == 0xEF) {
            if (thing->prev == NULL) {
                *list = thing->next;
            } else {
                thing->prev->next = thing->next;
            }
            if (thing->next != NULL) {
                thing->next->prev = thing->prev;
                thing->next = NULL;
            }
            thing->prev = NULL;
            MultiplyMatrix(&thing->mtx, mtx, &out);
            thing->mtx = out;
            thing->next = g_thingManager_DARYL.activeList;
            thing->prev = NULL;
            if (thing->next != NULL) {
                thing->next->prev = thing;
            }
            g_thingManager_DARYL.activeList = thing;
            if (thing->gridCell != 0 && (u8 *)thing->vtable == g_itemVTable) {
                Thing_AddToGrid(thing, 3);
                thing->flags &= ~8;
            }
        }
        thing = next;
    }
}
```

## PC match

```
0x0044a9c0: mov eax, [esp + 8]          ; list  (mtx = [esp + 4])
0x0044a9c4: sub esp, 0x40               ; Matrix4 out (64 bytes)
0x0044a9c7: push ebp
0x0044a9c8: mov ebp, [eax]              ; thing = *list
0x0044a9ca: test ebp, ebp
0x0044a9cc: je  0x44aa4a                ; empty -> exit, skipping ebx/esi/edi pushes+pops
0x0044a9ce: push ebx
0x0044a9cf: push esi
0x0044a9d0: push edi
0x0044a9d1: cmp byte ptr [ebp + 0x83], 0xef   ; slavePlayer == 0xEF?
0x0044a9d8: mov ebx, [ebp + 0xa4]       ; next (executed on BOTH paths)
0x0044a9de: jne 0x44aa41
0x0044a9e0: mov ecx, [esp + 0x58]       ; list (stack reload)
0x0044a9e4: push ebp
0x0044a9e5: push ecx
0x0044a9e6: call 0x44a7c0               ; ThingManagerList_Remove(list, thing)
0x0044a9eb: mov eax, [esp + 0x5c]       ; mtx (stack reload, push-shifted)
0x0044a9ef: lea edx, [esp + 0x18]       ; &out
0x0044a9f3: push edx
0x0044a9f4: push eax
0x0044a9f5: lea edi, [ebp + 0x10]       ; &thing->mtx (doubles as rep movsd dst)
0x0044a9f8: push edi
0x0044a9f9: call 0x4537c0               ; MultiplyMatrix(&thing->mtx, mtx, &out)
0x0044a9fe: push ebp                    ; Append arg (thing) staged BEFORE the copy
0x0044a9ff: mov ecx, 0x10
0x0044aa04: lea esi, [esp + 0x28]       ; same &out, re-based for 4 more pushes
0x0044aa08: push 0x53a9d0               ; &g_thingManager->activeList
0x0044aa0d: rep movsd                   ; thing->mtx = out (0x40 bytes)
0x0044aa0f: call 0x44a790               ; ThingManagerList_Append(&activeList, thing)
0x0044aa14: add esp, 0x1c               ; batched: Remove(2) + MultiplyMatrix(3) + Append(2)
0x0044aa17: cmp word ptr [ebp + 0x80], 0     ; gridCell != 0?
0x0044aa1f: je  0x44aa41
0x0044aa21: cmp dword ptr [ebp + 0x8c], 0x47aed8  ; vtable == g_itemVTable?
0x0044aa2b: jne 0x44aa41
0x0044aa2d: push 3
0x0044aa2f: push ebp
0x0044aa30: call 0x449de0               ; Thing_AddToGrid(thing, 3)
0x0044aa35: mov eax, [ebp + 0x74]
0x0044aa38: add esp, 8
0x0044aa3b: and eax, 0xfffffff7         ; flags &= ~8
0x0044aa3e: mov [ebp + 0x74], eax
0x0044aa41: test ebx, ebx
0x0044aa43: mov ebp, ebx
0x0044aa45: jne 0x44a9d1                ; re-enters AFTER the three pushes
0x0044aa47: pop edi
0x0044aa48: pop esi
0x0044aa49: pop ebx
0x0044aa4a: pop ebp
0x0044aa4b: add esp, 0x40
0x0044aa4e: ret
```

## Version differences

* **Unlink and append: inline vs call.** GC inlines both the prev/next
  surgery and the `activeList` push (the source above); PS2 inlines both as
  well — unlink @`0x002B14B8`–`0x002B14F0` (head store `sw $17,0x0($18)` in
  a `b` delay slot @`0x002B14C8`; `thing->next = NULL` @`0x002B14EC`),
  push @`0x002B1540`–`0x002B155C`. PC *calls* `ThingManagerList_Remove`
  @`0x0044A9E6` and `ThingManagerList_Append` @`0x0044AA0F`. Size follows:
  GC `0x158`, PS2 `0x13C`, PC `0x8F`.
* **`activeList` spelled two ways on PS2.** The push loads the head through
  the manager base (`lui`/`addiu D_66A030` + `lui`/`ori 0x7AB70`, `addu`
  @`0x002B1504`–`0x002B1520`, `lw` @`0x002B153C`) but *stores* the new head
  absolutely: `lui $1,0x6E4BA0` / `sw $16,0x4BA0($1)` @`0x002B1558`–
  `0x002B155C`. Same field, two spellings — independent confirmation that
  `activeList = 0x006E4BA0 = D_66A030 + 0x7AB70` (cf. `_README.md` bases:
  PC `0x0053A9D0`, GC bss `g_thingManager_DARYL`). PC hardcodes the literal
  `push 0x53a9d0` @`0x0044AA08`.
* **Matrix transform identical**: all three call
  `MultiplyMatrix(&thing->mtx, mtx, &out)` then copy the 64-byte `Matrix4`
  back into `thing+0x10`. PS2 args `addiu $4,$16,0x10`/`daddu $5,$19`/
  `addiu $6,$29,0x50` @`0x002B14F0`–`0x002B14F8`, `jal` @`0x002B14FC`; copy
  back with 4× `lq` from `sp+0x50` @`0x002B1510`–`0x002B1528` + 4× `sq` into
  `thing+0x10/20/30/40` @`0x002B152C`–`0x002B1538` (R5900 128-bit moves). PC
  `call 0x4537c0` @`0x0044A9F9` + `rep movsd` (ecx `0x10`) @`0x0044AA0D`; GC
  is the plain struct assignment.
* **MSVC scheduling gem.** The `rep movsd` copy-back sits *between* the two
  argument pushes of the `Append` call (`push ebp` @`0x0044A9FE`,
  `push 0x53a9d0` @`0x0044AA08`, `call` @`0x0044AA0F`), and `out` is
  addressed twice under different push-shifted offsets (`[esp+0x18]`
  @`0x0044A9EF`, `[esp+0x28]` @`0x0044AA04` — same address). Note the
  `rep movsd` destination `edi` was staged at `0x0044A9F5`, *before* the
  `MultiplyMatrix` call: MSVC's cdecl contract clobbers only
  `eax`/`ecx`/`edx`, so `0x4537C0` is guaranteed `edi`-preserving — useful
  when decompiling that helper.
* **Argument caching.** PS2 keeps `mtx` in `$19` for the whole loop; PC
  re-reads both arguments from the stack every iteration (`list`
  @`0x0044A9E0`, `mtx` @`0x0044A9EB`) and never caches them in callee-saved
  registers; GC keeps them as frame locals.
* **PS2 delay-slot staging everywhere.** `next` in the `bne` delay slot of
  the `0xEF` test @`0x002B14B4` (`lbu 0x83` + `addiu $3,$0,0xEF` compare);
  `thing->prev = NULL` (`sw $0,0xA0($16)`) folded into the `jal
  MultiplyMatrix` delay slot @`0x002B1500`; in the grid gate, `Thing_AddToGrid`'s
  `this` argument rides the *vtable-`bne` delay slot* @`0x002B157C`
  (`daddu $4,$16`) and `3` rides the `jal` delay slot @`0x002B1584` — both
  args staged without dedicating an instruction.
* **Grid gate**: `gridCell` read as a 16-bit field on all three — PS2 `lh
  $3,0x80` @`0x002B1560`, PC `cmp word ptr [ebp+0x80],0` @`0x0044AA17`;
  vtable literal `D_002C8FE0` (PS2 `g_itemVTable`) @`0x002B1570` vs
  `0x47AED8` @`0x0044AA21` vs GC symbol. `flags &= ~8` is a single statement
  everywhere: PS2 `addiu $3,$0,-0x9` (= `~8`) + `and` @`0x002B158C`–
  `0x002B1590`, PC `and eax,0xfffffff7` @`0x0044AA3B`.
* **Frames**: PC reserves exactly one `Matrix4` (`sub esp,0x40`
  @`0x0044A9C4`, torn down by `add esp,0x40` @`0x0044AA4B`) and uses the
  guarded-push idiom (`je 0x44aa4a` skips `push ebx/esi/edi` and their pops);
  PS2 uses a `0x90`-byte frame with `out` at `sp+0x50` and deallocates in the
  `jr $31` delay slot.
* **Semantics for the PC decomp**: `0xEF` is an emancipate sentinel written
  into `slavePlayer` (`0x83`) by whoever wants a slave promoted; slaves from
  `CreateSlave` exist on no global list until this runs, so this function is
  their sole join into `activeList`; grid re-insertion (`type 3`) with the
  in-grid flag cleared means "the world moved this item — rehash its cell".
