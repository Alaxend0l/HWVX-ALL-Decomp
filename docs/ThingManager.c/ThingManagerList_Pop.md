# `ThingManagerList_Pop` (coined — PC-only helper)

Removes and returns the head of `*list`, or returns `NULL` when the list is
empty. GC and PS2 have **no such function**: the identical sequence exists
twice, inlined inside `ThingManager_DARYL_CreateThing` and
`ThingManager_DARYL_CreateSlave` (`HWVX-GC-Decomp/src/ThingManager.c:1403`–
`1413` and `:1428`–`1438`). The PC linker factored it into one callable
helper. The name is ours — there is no GC/PS2 symbol to inherit; keep it
prefixed `ThingManagerList_` so it reads as part of the list cluster.

## Signature

* **GC** inline expansion at `src/ThingManager.c:1403` and `:1428` (no symbol).
* **PS2** inline expansion at `0x002AD614`–`0x002AD62C` (CreateThing) and
  `0x002AD578`–`0x002AD590` (CreateSlave) (no symbol).
* **PC** `void *ThingManagerList_Pop(void **list)` — `0x0044A810`, extent
  `0x30` (body ends `ret` @`0x0044A83B`), in unit `text_0044A0D0`; called
  from `ThingManager_DARYL_CreateThing` (`call` @`0x0044D0BD`) and
  `ThingManager_DARYL_CreateSlave` (`call` @`0x0044D0FD`).

## Arguments

`list` = address of a list-head slot (in practice `&g_thingManager.freeList[type]`).

## Execution

```c
void *ThingManagerList_Pop(void **list) {
    void *head = *list;

    if (head == NULL) {
        return NULL;
    }
    *list = M2C_FIELD(head, void **, 0xA4);            /* head = head->next   */
    if (*list != NULL) {
        M2C_FIELD(*list, void **, 0xA0) = NULL;        /* newhead->prev = NULL */
    }
    M2C_FIELD(head, void **, 0xA4) = NULL;             /* head->next = NULL    */
    return head;
}
```

This matches the GC inline expansion **exactly, quirks included**: the
redundant `if (head == NULL) head = NULL;` of source line 1405–1406 compiles
away everywhere, and `head->prev` (`0xA0`) is deliberately **not** cleared —
the popped thing keeps a stale `prev`. `CreateThing` repairs it with the
following Append; `CreateSlave` never does (harmless while the slave hangs on
its host's child list, and the sweep overwrites both links on free).

## PC match

```
0x0044a810: mov edx, [esp + 4]        ; list
0x0044a814: mov eax, [edx]            ; head
0x0044a816: test eax, eax
0x0044a818: jne 0x44a81b
0x0044a81a: ret                       ; NULL path — eax already 0
0x0044a81b: mov ecx, [eax + 0xa4]     ; head->next
0x0044a821: test ecx, ecx             ; flags survive the next store
0x0044a823: mov [edx], ecx            ; *list = next
0x0044a825: je  0x44a831
0x0044a827: mov [ecx + 0xa0], 0       ; newhead->prev = NULL
0x0044a831: mov [eax + 0xa4], 0       ; head->next = NULL
0x0044a83b: ret                       ; head in eax
```

Both callers materialise the slot address the same way —
`lea ecx,[eax*4+0x53a9d4]` @`0x0044D0B4` / @`0x0044D0F4` — i.e.
`&freeList[type]` with `freeList[]` at PC `0x0053A9D4` (= manager base
`+0x7AB74`, see `_README.md`).

## Version differences

* **The factoring is the difference.** GC/PS2 duplicate the pop in both
  creators; PC calls one helper twice. When matching PC, treat the two call
  sites as the GC inline blocks — do not look for a GC/PS2 symbol.
* **Loop-schedule details differ but semantics are identical**: PS2 stages
  `newhead->prev = NULL` in a guarded delay slot inside the inlined copies
  (`beqz`/`sw $0,0xA0($2)` @`0x002AD620`+`0x002AD628`), GC source reads like
  the plain sequence above; PC hoists the `test ecx,ecx` above the `*list`
  store (@`0x0044A821`/`0x0044A823`) — MSVC again, flags survive `mov`.
* **NULL-return materialisation**: PC's empty-list path returns whatever the
  head load left in `eax` (0) with no extra instruction (`ret` @`0x0044A81A`);
  the PS2 inline copies stage `daddu $16,$0,$0` in a branch delay slot
  (@`0x002AD610` / `0x002AD574`) because the popped pointer lives in a
  callee-saved register.
* GC's CreateThing inline copy re-reads `freeList[type]` after storing
  (`src` line 1409 `if (g_thingManager_DARYL.freeList[type] != NULL)`, PS2
  `lw $2,0x0($3)` @`0x002AD61C` re-loading the just-stored head); the PC
  helper instead tests the **cached** `head->next` register
  (`test ecx,ecx` @`0x0044A821`) — same value, different form.
