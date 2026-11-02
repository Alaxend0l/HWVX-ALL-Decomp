# `ThingManagerList_Append`

Pushes `thing` at the **head** of `*list` (the name says "Append", the
semantics are prepend — all three builds).

## Signature

* **GC** `void ThingManagerList_Append(void **arg0, void *arg1)` —
  `0x80053F78`, size `0x28` (`HWVX-GC-Decomp/src/ThingManager.c:5`).
* **PS2** — `0x002B1910`, size `0x24`
  (`asm/nonmatchings/cod/ThingManagerList_Append.s`).
* **PC** — `0x0044A790`, extent `0x30` (body `0x27` bytes + `int3` padding),
  in unit `text_0044A0D0`.

## Arguments

`arg0` = address of a list-head slot (`&manager.activeList`,
`&manager.freeList[type]`, or `&thing->unk88` for private child lists);
`arg1` = the thing being linked in. Uses only `ThingHeader` fields `0xA0`
(`prev`) and `0xA4` (`next`).

## Execution

```c
void ThingManagerList_Append(void **arg0, void *arg1) {
    M2C_FIELD(arg1, void **, 0xA4) = (void *)*arg0;   /* thing->next = head   */
    M2C_FIELD(arg1, s32 *, 0xA0) = 0;                 /* thing->prev = NULL   */
    if (M2C_FIELD(arg1, void **, 0xA4) != NULL) {
        M2C_FIELD(M2C_FIELD(arg1, void **, 0xA4), void **, 0xA0) = arg1;
                                                      /* head->prev = thing   */
    }
    *arg0 = arg1;                                     /* head = thing         */
}
```

No bounds/null checks; the caller guarantees `arg0` points at a live head.

## PC match

```
0x0044a790: mov edx, [esp + 4]        ; arg0
0x0044a794: mov ecx, [edx]            ; head
0x0044a796: test ecx, ecx             ; flags live through the next movs
0x0044a798: mov eax, [esp + 8]        ; arg1
0x0044a79c: mov [eax + 0xa4], ecx     ; thing->next = head      (line 6)
0x0044a7a2: mov [eax + 0xa0], 0       ; thing->prev = NULL      (line 7)
0x0044a7ac: je  0x44a7b4
0x0044a7ae: mov [ecx + 0xa0], eax     ; head->prev = thing      (line 9)
0x0044a7b4: mov [edx], eax            ; *arg0 = arg1            (line 11)
0x0044a7b6: ret
```

Field offsets `0xA0`/`0xA4` and the store order match the GC source exactly.

## Version differences

* **NULL-test source**: GC line 8 *re-reads* `arg1->next` and PS2 re-loads it
  (`lw $3,0xA4($5)` @`0x002B191C`) even though it was stored one instruction
  earlier; PC instead tests the cached head register — `test ecx,ecx`
  @`0x0044A796`, hoisted *above* both stores, with the `je` issued late
  (@`0x0044A7AC`, after the `0xA0` store). Same value on all three; the PC
  form is MSVC scheduling, not a semantic difference.
* **PS2 delay-slot folding**: only the final `*arg0 = arg1` store is folded
  into the return delay slot (`jr $31` / `sw $5,0x0($4)`
  @`0x002B192C`/`0x002B1930`) — the head update is *unconditional last*,
  exactly like the source. The guarded `head->prev = thing` store is a plain
  instruction skipped by the `beqz` (`0x002B1928`); the branch's delay slot is
  a wasted `nop` (`0x002B1924`), as is the one after the `jr` (`0x002B1934`).
* **Size**: PS2 `0x24` < GC `0x28` < PC extent `0x30` (PC body is only
  `0x27`; the extent includes alignment padding).
