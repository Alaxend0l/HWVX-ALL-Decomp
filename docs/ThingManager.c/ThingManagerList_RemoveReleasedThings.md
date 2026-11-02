# `ThingManagerList_RemoveReleasedThings`

Sweep pass over a thing list: for every node flagged `0x200000`
(pending-free), first release-and-sweep its private child list (`+0x88`),
then unlink the node from `list`, push it onto the manager free list for
its `type`, and rewrite `flags` (`&= ~0x200000`, `|= 0x1000`). Recurses
through `ThingManagerList_ReleaseAndRemoveAll` (GC/PC) so arbitrarily deep
slave trees are torn down in one call.

## Signature

* **GC** `void ThingManagerList_RemoveReleasedThings(void **list)` —
  `0x8005414C`, size `0xF8` (`HWVX-GC-Decomp/src/ThingManager.c:166`).
* **PS2** — `0x002B1610`, size `0x12C`
  (`asm/nonmatchings/cod/ThingManagerList_RemoveReleasedThings.s`).
* **PC** — `0x0044A910`, extent `0x60` (body ends `ret` @`0x0044A96F`), in
  unit `text_0044A0D0`.

## Arguments

`list` = pointer to a list-head slot. No return value.

## Execution

```c
void ThingManagerList_RemoveReleasedThings(void **list) {
    void ThingManagerList_ReleaseAndRemoveAll(void *list);
    ThingHeader *thing;
    ThingHeader *next;

    for (thing = *list; thing != NULL; thing = next) {
        next = thing->next;
        if (thing->flags & 0x200000) {
            ThingManagerList_ReleaseAndRemoveAll(&thing->unk88);
            ThingManagerList_Remove(list, thing);
            ThingManagerList_Append(&g_thingManager_DARYL.freeList[thing->type], thing);
            thing->flags &= ~0x200000;
            thing->flags |= 0x1000;
        }
    }
}
```

(The local prototype declaration is part of the decompiled shape — GC
places it inside the function body, see `src/ThingManager.c:167`.)

## PC match

```
0x0044a910: push ebx
0x0044a911: mov ebx, [esp + 8]          ; list
0x0044a915: push esi
0x0044a916: mov esi, [ebx]              ; thing = *list
0x0044a918: test esi, esi
0x0044a91a: je  0x44a96d                ; empty -> exit, skipping edi push/pop
0x0044a91c: push edi
0x0044a91d: test dword ptr [esi + 0x74], 0x200000
0x0044a924: mov edi, [esi + 0xa4]       ; next (executed on BOTH paths)
0x0044a92a: je  0x44a966
0x0044a92c: lea eax, [esi + 0x88]
0x0044a932: push eax
0x0044a933: call 0x44a970               ; ReleaseAndRemoveAll(&thing->unk88)
0x0044a938: push esi
0x0044a939: push ebx
0x0044a93a: call 0x44a7c0               ; Remove(list, thing)
0x0044a93f: mov ecx, [esi + 0xa8]
0x0044a945: lea edx, [ecx*4 + 0x53a9d4] ; &freeList[type]
0x0044a94c: push esi
0x0044a94d: push edx
0x0044a94e: call 0x44a790               ; Append(&freeList[type], thing)
0x0044a953: mov eax, [esi + 0x74]
0x0044a956: and eax, 0xffdfffff
0x0044a95b: add esp, 0x14               ; batched cleanup wedged BETWEEN the two flag ops
0x0044a95e: or  eax, 0x1000
0x0044a963: mov [esi + 0x74], eax
0x0044a966: test edi, edi
0x0044a968: mov esi, edi
0x0044a96a: jne 0x44a91d                ; re-enters AFTER push edi
0x0044a96c: pop edi
0x0044a96d: pop esi
0x0044a96e: pop ebx
0x0044a96f: ret
```

## Version differences

* **PS2 folds in the recursion target.** PS2 has **no**
  `ThingManagerList_ReleaseAndRemoveAll` symbol (see that doc); this
  function absorbs its job. When `thing->unk88 != NULL` it runs an
  **inlined `ReleaseAll` walk** over the children — `lw $2,0x8C($4)` /
  `lw $2,4($2)` / `jalr $2` @`0x2B1658`–`0x2B1660` with next captured in
  the delay slot (`lw $19,0xA4($4)` @`0x2B1664`), loop `bnez` @`0x2B1668` —
  and *then* `jal ThingManagerList_RemoveReleasedThings` **itself**
  @`0x2B1670` with the delay slot `addiu $4,$16,0x88` @`0x2B1674`
  (`$4 = &thing->unk88`). Net semantics are identical to GC's
  `ReleaseAndRemoveAll(children)`: release-everything then sweep. Not a
  behavioral divergence — a codegen merge.
* **Even an empty child list takes the call.** The PS2 guard
  `beqz $4,.L002B1670` @`0x2B164C` skips only the inline walk, not the
  self-`jal`: with `unk88 == NULL` the recursion still executes and the
  callee's own empty-list test (`beqz $16` @`0x2B162C`) no-ops it. GC/PC
  skip the child call entirely when the sweep's per-node test fails, but
  still call into `ReleaseAndRemoveAll` unconditionally once flagged —
  same "call an empty list, callee checks" contract.
* **Inlining profile.** PS2 additionally inlines Remove
  (@`0x2B1678`–`0x2B16B0`, head store in a `b` delay slot @`0x2B168C`,
  with the full reload quirk) and Append (@`0x2B16B4`–`0x2B16F0`, `lbu`
  type @`0x2B16B4`, store to `D_66A030 + 0x7AB74` slot @`0x2B16F0`) and
  does the flag pair as two read-modify-writes
  (@`0x2B16F4`–`0x2B1710`). GC calls Remove/Append. PC calls the wrapper +
  Remove + Append → sizes: GC `0xF8`, PS2 `0x12C` (biggest), PC `0x60`.
* **`next` placement.** GC captures before the flag test (source line 2);
  PS2 in the flag-test `beqz` delay slot @`0x2B1644`; PC between `test`
  and `je` @`0x0044A924`. All three execute it on both paths, so `next`
  survives the removal.
* **PC op order matches the source exactly** — recurse → Remove → Append →
  `&= ~0x200000` → `|= 0x1000` — with MSVC wedging the batched
  `add esp,0x14` @`0x0044A95B` *between* the `and` and the `or`
  (@`0x0044A956`/`0x0044A95E`), the opposite fusion direction from
  `ReleaseSlavesBySlaveType` (where the two ops were merged). Flag-safe
  scheduling again: `add esp` does not touch flags or `eax`.
* Guarded `push edi` @`0x0044A91C`/`pop edi` @`0x0044A96C` with the loop
  re-entering at `0x0044A91D` — same once-only-push idiom as
  `ReleaseSlavesBySlaveType`.
