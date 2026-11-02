# `ThingManagerList_ReleaseAndRemoveAll`

Composite teardown of a list: first `ThingManagerList_ReleaseAll` (call every
node's vtable `Release` slot), then the standard sweep — every node flagged
`0x200000` gets its private child list torn down recursively, is unlinked from
`list`, pushed onto its `type` free list, and has `flags` rewritten
(`&= ~0x200000`, `|= 0x1000`). Used for whole-list destruction (manager
teardown: `src/ThingManager.c:656`–`:657` runs it twice on `activeList`).

## Signature

* **GC** `void ThingManagerList_ReleaseAndRemoveAll(void **list)` —
  `0x80054244`, size `0x228` (`HWVX-GC-Decomp/src/ThingManager.c:184`, with a
  forward prototype of `ReleaseAll` on line `:183`).
* **PS2** — **no symbol exists** (see below and `_README.md` "PS2 symbol
  gaps"); the job is compiled into `ThingManagerList_RemoveReleasedThings`.
* **PC** — `0x0044A970`, extent `0x20` (body ends `ret` @`0x0044A985`), in
  unit `text_0044A0D0`.

## Arguments

`list` = pointer to a list-head slot. No return value.

## Execution

```c
void ThingManagerList_ReleaseAll(void **list);

void ThingManagerList_ReleaseAndRemoveAll(void **list) {
    ThingHeader *thing;
    ThingHeader *next;

    ThingManagerList_ReleaseAll(list);
    for (thing = *list; thing != NULL; thing = next) {
        next = thing->next;
        if ((thing->flags & 0x200000) != 0) {
            void **children;
            ThingManagerList_ReleaseAndRemoveAll(children = (void **)&thing->unk88);
            ThingManagerList_Remove(list, thing);
            ThingManagerList_Append(g_thingManager_DARYL.freeList + thing->type, thing);
            thing->flags &= ~0x200000;
            thing->flags |= 0x1000;
        }
    }
}
```

The sweep body is *character-identical* to
`ThingManagerList_RemoveReleasedThings`'s (`:166`) except for the
`children = ...` assignment inside the call argument — a reconstruction
artifact of call-argument evaluation, not real code.

## PC match

```
0x0044a970: push esi
0x0044a971: mov esi, [esp + 8]          ; list
0x0044a975: push esi
0x0044a976: call 0x44a990               ; ThingManagerList_ReleaseAll(list)
0x0044a97b: push esi
0x0044a97c: call 0x44a910               ; ThingManagerList_RemoveReleasedThings(list)
0x0044a981: add esp, 8
0x0044a984: pop esi
0x0044a985: ret
```

A pure two-call wrapper — MSVC reused the existing sweep inside
`RemoveReleasedThings` instead of duplicating it.

## Version differences

* **PS2 has no such function.** The 8-byte slot between `ReleaseAll`'s last
  instruction (frame restore `addiu $29,$29,0x20` in the `jr $31` delay slot
  @`0x002B1604`) and `ThingManagerList_RemoveReleasedThings` @`0x002B1610`
  was read directly from `game.elf` at file offset `0x0B1708` (mapping
  `off = 0x100 + vaddr - 0x200000`, validated against known prologues at
  `0x002B15C0`/`0x002B1610`/`0x002B1910`): two zero words — the same `nop`s
  the `.s` listing prints after `endlabel`. Alignment padding, **not** a
  stub. The GC function's role is inlined into `RemoveReleasedThings`: an
  `ReleaseAll` walk over the child list (@`0x002B1654`–`0x002B166C`, a
  byte-clone of the real loop @`0x002B15D8`–`0x002B15EC` differing only in
  `$19`↔`$16` for `next`) followed by `jal` **itself** @`0x002B1670` with
  delay slot `addiu $4,$16,0x88` (`$4 = &thing->unk88`). Walk-then-sweep:
  net-equivalent to GC's `ReleaseAll(children)` + recursive sweep.
* **Recursion shape** differs on all three, semantics identical: GC is
  *self*-recursive inside `ReleaseAndRemoveAll` (sweep inlined); PC is
  *mutually* recursive (`RemoveReleasedThings` @`0x0044A933` calls this
  wrapper, which calls `RemoveReleasedThings` back); PS2 is *pure self*
  recursion in the merged `RemoveReleasedThings`. Termination is the same
  everywhere — an empty child list hits the callee's entry guard.
* **Code duplication vs sharing.** GC `0x228` = `ReleaseAll` call + the full
  sweep inlined, while `RemoveReleasedThings` (`0xF8`) already holds a second
  copy of that sweep — SN Systems duplicated it. MSVC kept the source
  structure shared: the PC wrapper is `0x16` bytes of call staging. This is
  the strongest single piece of evidence that the PC source matched the GC
  reconstruction and only the compilers' inlining decisions differed.
* **`children = &thing->unk88`** as a call argument: only the GC
  reconstruction exhibits the assignment (m2c recovered the value being both
  computed and passed); PS2 shows the same shape as the delay-slot
  `addiu $4,$16,0x88`; PC `lea eax,[esi+0x88]`/`push eax` @`0x0044A92C` in
  the caller-side sweep just passes the address.
