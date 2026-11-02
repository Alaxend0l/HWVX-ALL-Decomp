# `ThingManagerList_ReleaseSlavesBySlaveType`

Sweeps a slave list for entries whose `slavePlayer` byte (`0x83`) equals
`type`, and for each: calls the vtable `Release` slot, unlinks it from the
list, clears `0x200000` / sets `0x1000` in `flags` (`0x74`), and pushes it
onto the manager free list for its `type` (`0xA8`). The list compacts; the
released things return to the pool.

## Signature

* **GC** `void ThingManagerList_ReleaseSlavesBySlaveType(void **list, u8 type)`
  — `0x8005403C`, size `0x110` (`HWVX-GC-Decomp/src/ThingManager.c:139`).
* **PS2** — `0x002B1740`, size `0x10C`
  (`asm/nonmatchings/cod/ThingManagerList_ReleaseSlavesBySlaveType.s`).
* **PC** — `0x0044A8A0`, extent `0x70` (body ends `ret` @`0x0044A905`), in
  unit `text_0044A0D0`.

## Arguments

`list` = pointer to a list-head slot (typically a thing's private child
list at `thing+0x88`); `type` = `u8` matched against each node's
`slavePlayer` byte at `+0x83`.

## Execution

```c
void ThingManagerList_ReleaseSlavesBySlaveType(void **list, u8 type) {
    void *thing;
    void *next;
    void **freeList;
    void *head;

    thing = *list;
    while (thing != NULL) {
        next = M2C_FIELD(thing, void **, 0xA4);
        if (M2C_FIELD(thing, u8 *, 0x83) == type) {
            M2C_FIELD(M2C_FIELD(thing, void **, 0x8C), void (**)(void *), 4)(thing);
            ThingManagerList_Remove(list, thing);
            M2C_FIELD(thing, u32 *, 0x74) &= ~0x200000;
            M2C_FIELD(thing, u32 *, 0x74) |= 0x1000;
            freeList = &g_thingManager_DARYL.freeList[M2C_FIELD(thing, s32 *, 0xA8)];
            M2C_FIELD(thing, void **, 0xA4) = *freeList;
            M2C_FIELD(thing, void **, 0xA0) = NULL;
            head = M2C_FIELD(thing, void **, 0xA4);
            if (head != NULL) {
                M2C_FIELD(head, void **, 0xA0) = thing;
            }
            *freeList = thing;
        }
        thing = next;
    }
}
```

## PC match

```
0x0044a8a0: push ebp
0x0044a8a1: mov ebp, [esp + 8]          ; list
0x0044a8a5: push esi
0x0044a8a6: mov esi, [ebp]              ; thing = *list
0x0044a8a9: test esi, esi
0x0044a8ab: je  0x44a903                ; empty -> exit, skipping ebx/edi pushes+pops
0x0044a8ad: push ebx
0x0044a8ae: mov bl, [esp + 0x14]        ; type (push-shifted arg)
0x0044a8b2: push edi
0x0044a8b3: cmp byte ptr [esi + 0x83], bl   ; slavePlayer == type?
0x0044a8b9: mov edi, [esi + 0xa4]       ; next (executed on BOTH paths)
0x0044a8bf: jne 0x44a8fb
0x0044a8c1: mov eax, [esi + 0x8c]       ; vtable
0x0044a8c7: push esi                    ; this
0x0044a8c8: call dword ptr [eax + 4]    ; vtable[1] Release(thing)
0x0044a8cb: push esi
0x0044a8cc: push ebp
0x0044a8cd: call 0x44a7c0               ; ThingManagerList_Remove(list, thing)
0x0044a8d2: mov ecx, [esi + 0x74]       ; flags — one read for BOTH ops
0x0044a8d5: mov edx, [esi + 0xa8]       ; thing->type
0x0044a8db: and ecx, 0xffdfffff         ; ~0x200000
0x0044a8e1: lea eax, [edx*4 + 0x53a9d4] ; &freeList[type]
0x0044a8e8: or  ecx, 0x1000
0x0044a8ee: push esi
0x0044a8ef: push eax
0x0044a8f0: mov [esi + 0x74], ecx       ; one store
0x0044a8f3: call 0x44a790               ; ThingManagerList_Append(&freeList[type], thing)
0x0044a8f8: add esp, 0x14               ; cleans ALL 5 arg pushes of the 3 calls
0x0044a8fb: test edi, edi
0x0044a8fd: mov esi, edi
0x0044a8ff: jne 0x44a8b3                ; re-enters AFTER push edi
0x0044a901: pop edi
0x0044a902: pop ebx
0x0044a903: pop esi
0x0044a904: pop ebp
0x0044a905: ret
```

`0x53A9D4` = PC manager base `0x004BFE60` + `0x7AB74` = `freeList[0]`
(see `_README.md` pool table).

## Version differences

* **Who calls whom.** GC source *calls* `ThingManagerList_Remove` but
  inlines the free-list push; PS2 **inlines both** the Remove body
  (@`0x002B1788`–`0x002B17C0`, head store in a `b` delay slot @`0x002B179C`)
  and the Append body (@`0x002B1804`–`0x002B1820`); PC **calls both**
  (`0x0044A8CD`, `0x0044A8F3`). Size follows: GC `0x110`, PS2 `0x10C`
  (inlining keeps it big), PC `0x70` (calls make it tiny).
* **Flag update fusion.** GC/PS2 emit the two source statements as two
  read-modify-writes — PS2 `and`+store @`0x002B17E0`/`0x002B17E4` then a
  full *reload* `lw $4,0x74($16)` + `ori 0x1000` + store; PC fuses to one
  load @`0x0044A8D2`, `and` @`0x0044A8DB`, `or` @`0x0044A8E8`, one store
  @`0x0044A8F0`. (Interleaved between them, MSVC computes the free-list
  slot `lea` @`0x0044A8E1` and stages the Append pushes.)
* **`next` capture placement.** All three capture `next` at loop top before
  the match branch so the walk survives the unlink: GC source line 1; PS2
  folds `lw $17,0xA4($16)` into the *delay slot of the slavePlayer test*
  `bne` @`0x002B1774` (delay slots run on both taken and not-taken, so one
  copy covers both paths); PC `mov edi,[esi+0xa4]` @`0x0044A8B9` sits
  between the `cmp` and the `jne` — also both paths.
* **`u8` narrowing.** PS2 masks the argument (`andi $3,$18,0xFF`
  @`0x002B176C`) and loads the field with `lbu $4,0x83($16)`; PC uses byte
  operands (`mov bl,[esp+0x14]` @`0x0044A8AE`, `cmp byte ptr [esi+0x83],bl`
  @`0x0044A8B3`); GC's `u8 type` parameter sign-extends per the ABI but
  the compare is against a byte field everywhere. PS2's free-list index
  likewise uses `lbu $4,0xA8` @`0x002B1804` where PC reads the dword
  `mov edx,[esi+0xa8]` @`0x0044A8D5`.
* **MSVC batched cleanup / guarded pushes.** One `add esp,0x14`
  @`0x0044A8F8` cleans the args of all three calls (the vtable call's
  `push esi` is NOT cleaned after `call [eax+4]` @`0x0044A8C8` — it is
  counted here). `push edi` @`0x0044A8B2` executes once: the loop's `jne`
  @`0x0044A8FF` re-enters at `0x0044A8B3`, past it; the empty-list early
  exit @`0x0044A8AB` jumps to `0x0044A903`, skipping both the ebx/edi
  pushes and their pops @`0x0044A901`–`0x0044A902`.
* **Loop continuation.** PS2: `bnez $17,.L002B1768` @`0x002B1828` with
  `daddu $16,$17,$0` in the delay slot; PC: `test edi,edi` / `mov esi,edi`
  / `jne` @`0x0044A8FB`–`0x0044A8FF`. Same shape, `$17`↔`edi`.
