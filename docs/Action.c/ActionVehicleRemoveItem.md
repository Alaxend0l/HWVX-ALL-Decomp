# `ActionVehicleRemoveItem`

## Signature

* **GC** `s32 ActionVehicleRemoveItem(u32 *action)` — `0x8000659C`, size `0xCC`
  (`HWVX-GC-Decomp/src/Action.c:561`).
* **PS2** `s32 ActionVehicleRemoveItem(s32 *action)` — `0x0024E640`, size `0xFC`
  (`asm/nonmatchings/cod/ActionVehicleRemoveItem.s`; no m2c draft).
* **PC** `s32 __cdecl ActionVehicleRemoveItem(u32 *action)` — `0x00401A00`, size `0x90`
  (unit `text_00401090`; entry `push ecx; push ebx` at `0x00401A00`, code to `0x00401A8C`,
  3 bytes nop padding). **PC-CONFIRMED** (full disassembly verified).

## Arguments

`action` = record's `data`: `action[0]` = mode (`0` jettison rack, `1` drop one item),
`action[1]` = list length, `action[2 + i]` = vehicle pointer whose `+0x28` is the creature.
The creature's thing list (`+0x344` GC/PS2, **`+0x324` PC**) holds the wildcard **rack**
(type 8) and carried **item** things (type 4).

## Execution

```c
s32 ActionVehicleRemoveItem(u32 *action) {
    /* ItemThing { flags@0x74, vtable@0x8C } */
    for (i = 0; i < action[1]; i++) {
        creature = ((Vehicle *)action[i + 2])->creature;    /* +0x28 */
        if (creature != NULL) {
            list = creature + 0x344;
            rack = ThingManagerList_GetByType(list, 8);
            switch (action[0]) {
            case 0:
                WildCardRack_JettisonAll(rack);             /* unguarded call */
                break;
            case 1:
                item = ThingManagerList_GetByType(list, 4); /* SECOND lookup */
                if (item != NULL) {
                    item->vtable[1](item);                  /* virtual drop */
                }
                break;
            }
            rack->flags &= ~0x400200;                       /* UNGUARDED rack */
        }
    }
    return 1;
}
```

## PC match

`0x00401A00` — `push ecx; push ebx; mov ebx,[esp+0xc]`.

* `mov eax,[ebx+4]` (`action[1]`), `mov [esp+4],0` (loop var `i` in a **stack slot**),
  `test eax,eax; jbe 0x00401A88` — count 0 returns 1 immediately (matching an empty loop).
* `lea ebp,[ebx+8]` = `&action[2]`; creature `[eax+0x28]` guarded;
  `lea esi,[eax+0x324]` (PC thing-list offset); `push 8; push esi; call 0x0044A840`
  (GetByType) @`0x00401A2E` → `edi` = rack.
* Mode cascade (source order): `sub eax,0; je 0x00401A5D` → mode 0: `push edi;
  call 0x00451C00` (JettisonAll) **direct and unguarded** @`0x00401A5E`;
  `dec eax; jne 0x00401A66` → mode 1 @`0x00401A42`: `push 4; push esi;
  call 0x0044A840` @`0x00401A45`, guard `test eax,eax; je 0x00401A66`, then
  `mov ecx,[eax+0x8c]; push eax; call dword ptr [ecx+4]` — **`item->vtable[1](item)`**
  (vtable field `+0x8C`, slot `+0x04`) ✓ GC 592.
* Shared arm-join @`0x00401A66`: `and dword ptr [edi+0x74],0xFFBFFDFF` — clears bits
  `0x00000402 | ... ` i.e. `~0x400200`, **unguarded** on the rack ✓ GC 596.
* Loop tail re-reads `action[1]` (`mov ecx,[ebx+4]`); `mov al,1; pop ebx; pop ecx; ret`.

## Version differences

* **Clean 3-way match** — the logic (two GetByType calls on the same list, direct unguarded
  `WildCardRack_JettisonAll`, guarded virtual `vtable[1]` drop, unguarded
  `rack->flags &= ~0x400200`, unconditional `return 1`) is identical on GC, PS2 and PC.
* **List offset**: creature `+0x344` (GC: `list = creature + 0x344`; PS2
  `addiu $4,$x,0x344`) vs PC `+0x324` — the known CarA thing-list layout skew.
* **Mode cascade order**: PC tests **mode 0 first** (`sub eax,0; je`) then mode 1
  (`dec; jne`); PS2 tests **mode 1 first** (`beq` → `.L0024E6C0` @`0x0024E6C4`-ish, then
  `beqz` → `.L0024E6AC` @`0x0024E6B0`) — the PS2 SN compiler reordered the arms; both skip
  correctly to the shared flag-clear.
* **Flag clear**: PS2 needs the three-instruction `ori 0xFFBFFDFF`/`and`/`sw 0x74($17)`
  idiom @`0x0024E6E8`–`0x0024E6F8` (immediate doesn't fit `andi`); PC does it in one
  `and [edi+0x74],0xffbffdff` @`0x00401A66`; GC draft the single `&= ~0x400200`.
* **PC prologue quirk**: loop counter `i` lives in a **stack slot** `[esp+4]` (MSVC /G5
  spill: `mov [esp+4],0` / reload `mov eax,[esp+0x10]` after in-loop pushes) while the
  in-loop `push ebp/esi/edi` stay balanced — the early `jbe 0x00401A88` exit only pops
  `ebx/ecx`. PS2/GC use callee-saved registers; no semantic difference.
* Count re-read per iteration on all platforms (PC `mov ecx,[ebx+4]` @`0x00401A71`, PS2
  `lw $2,0x4($19)`) — scripts can mutate the list length mid-loop.
* PS2 mode re-read from `lw $3,0($19)` per vehicle (mode not hoisted); GC draft re-reads
  `action[0]` per iteration too.
