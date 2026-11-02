# `ThingManager_DARYL_CreateSlave`

Allocates a thing from `freeList[type]` as a **slave**: runs its class init
and stamps it with a trigger id + slave flag, but does **not** put it on the
global active list. Slaves hang on their host's private child list
(`ThingHeader+0x88`) and only join the active list when
`ThingManagerList_EmancipateSlaves` promotes them.

## Signature

* **GC** `void *ThingManager_DARYL_CreateSlave(s32 type, void *params, u8 id)`
  — `0x80057CE0`, size `0xB4` (`HWVX-GC-Decomp/src/ThingManager.c:1427`).
* **PS2** — `0x002AD540`, size `0x98`
  (`asm/nonmatchings/cod/ThingManager_DARYL_CreateSlave.s`).
* **PC** — `0x0044D0F0`, extent `0x50` (unit `text_0044D050`).

## Arguments

`type`/`params` as in `ThingManager_DARYL_CreateThing.md`; `id` is stored as
`ThingHeader.slaveId` (`+0x82`) — the trigger id `ThingManagerList_TriggerSlaves`
and `ThingManagerList_ReleaseSlavesBySlaveType` later match against
(also `+0x83` `slavePlayer`, written by callers, `0xEF` = emancipate marker).

## Execution

```c
void *ThingManager_DARYL_CreateSlave(s32 type, void *params, u8 id) {
    void *slave = g_thingManager_DARYL.freeList[type];

    if (slave == NULL) {
        slave = NULL;                     /* redundant, same as CreateThing */
    } else {                              /* --- inlined Pop(freeList[type]) */
        g_thingManager_DARYL.freeList[type] = M2C_FIELD(slave, void **, 0xA4);
        if (g_thingManager_DARYL.freeList[type] != NULL) {   /* re-read! */
            M2C_FIELD(g_thingManager_DARYL.freeList[type], void **, 0xA0) = NULL;
        }
        M2C_FIELD(slave, void **, 0xA4) = NULL;
    }
    if (slave != NULL) {
        ((void (*)(void *, void *))(M2C_FIELD(slave, void ***, 0x8C))[0])(slave, params);
        M2C_FIELD(slave, u8 *, 0x82) = id;                  /* slaveId    */
        M2C_FIELD(slave, u32 *, 0x74) |= 0x20000000;        /* slave flag */
        return slave;
    }
    return NULL;
}
```

Contrast with `CreateThing`: **no Append**, and init runs *before* the
id/flag stamping (so `Init` sees `slaveId == stale pool value` — classes must
not read it during init). The popped `slave->prev` stays stale (Pop quirk,
`ThingManagerList_Pop.md`); nothing repairs it while the slave is unlinked.

## PC match

```
0x0044d0f0: mov eax, [esp + 4]            ; type
0x0044d0f4: lea ecx, [eax*4 + 0x53a9d4]   ; &freeList[type]
0x0044d0fb: push esi
0x0044d0fc: push ecx
0x0044d0fd: call 0x44a810                 ; ThingManagerList_Pop
0x0044d102: mov esi, eax                  ; slave
0x0044d104: add esp, 4
0x0044d107: test esi, esi
0x0044d109: je  0x44d135                  ; exhausted -> NULL
0x0044d10b: mov eax, [esp + 0xc]          ; params
0x0044d10f: mov edx, [esi + 0x8c]         ; vtable
0x0044d115: push eax
0x0044d116: push esi
0x0044d117: call dword ptr [edx]          ; vtable[0](slave, params)
0x0044d119: mov eax, [esi + 0x74]         ; flags
0x0044d11c: mov cl, [esp + 0x18]          ; id (3rd arg, push-shifted)
0x0044d120: add esp, 8
0x0044d123: or eax, 0x20000000            ; slave flag
0x0044d128: mov [esi + 0x74], eax         ; flags stored FIRST
0x0044d12b: mov byte ptr [esi + 0x82], cl ; slaveId stored SECOND
0x0044d131: mov eax, esi
0x0044d133: pop esi
0x0044d134: ret
0x0044d135: xor eax, eax
0x0044d137: pop esi
0x0044d138: ret
```

## Version differences

* **Store order of the two stamps is swapped on PC.** GC lines 1441–1442 and
  PS2 write `slaveId` *then* OR the flag (`sb $17,0x82($16)` @`0x002AD5AC`
  before `lui`/`or`/`sw` @`0x002AD5B0`–`0x002AD5C0`); PC stores flags first
  (@`0x0044D128`) and the id byte after (@`0x0044D12B`). The fields are
  disjoint so the swap is invisible at runtime, but it is the fingerprint
  that separates this function from a hypothetical reordered rebuild.
* **PC fetches `id` after the init call returns** — `mov cl,[esp+0x18]`
  @`0x0044D11C` (entry slot `[esp+0xC]` shifted by `push esi`, `add esp,4`
  and the two init pushes). PS2 instead stages id into callee-saved `$17`
  inside the *first* branch's delay slot (`bnez` / `daddu $17,$6,$0`
  @`0x002AD568`/`0x002AD56C` — the delay slot executes unconditionally, so id
  is captured even on the empty-pool path where it is never used).
* **PS2 byte-narrows `type`**: `andi $3,$4,0xFF` @`0x002AD544` (same trap as
  `GetByType`/`CreateThing`).
* **Flag constant materialisation**: PC single `or eax,0x20000000`
  @`0x0044D123`; PS2 needs `lui $3,(0x20000000>>16)` @`0x002AD5B0` +
  `or` @`0x002AD5BC`, and stages the return value `daddu $2,$16,$0`
  @`0x002AD5B8` *between* the flag load and the OR; the NULL return is staged
  in the guard branch's delay slot (`beqz $16` / `daddu $2,$0,$0`
  @`0x002AD594`/`0x002AD598`).
* **Factoring**: PC calls `ThingManagerList_Pop` @`0x0044D0FD`; GC/PS2 inline
  the identical pop (PS2 block `0x002AD578`–`0x002AD590`, including the
  re-read `lw $2,0x0($3)` @`0x002AD580` mirroring GC's line-1431 reload).
* No `0x7AB70`/`activeList` reference anywhere in this function on any
  platform — the absence of the Append block is itself the structural
  fingerprint distinguishing it from `CreateThing`.
