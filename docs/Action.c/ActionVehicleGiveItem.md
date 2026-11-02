# `ActionVehicleGiveItem`

## Signature

* **GC** `s32 ActionVehicleGiveItem(u32 *action)` — `0x80006668`, size `0x39C`
  (`HWVX-GC-Decomp/src/Action.c:604`).
* **PS2** `s32 ActionVehicleGiveItem(s32 *action)` — `0x0024E280`, size `0x3BC`
  (`asm/nonmatchings/cod/ActionVehicleGiveItem.s`; no m2c draft).
* **PC** `s32 __cdecl ActionVehicleGiveItem(u32 *action)` — `0x00401A90`, size `0x360`
  (unit `text_00401090`; code to `0x00401DE2`, nop padding to `0x00401DF0`).
  **PC-CONFIRMED** (full disassembly verified).

The largest function in Action.c — the item-spawn/give path, and the module's best
PC fingerprint because of its `sub esp,0x8c` aligned-frame prologue.

## Arguments

`action` = record's `data`, read as `Action { u32 type; ItemInfo *item; u32 count;
void *cars[]; }` (`type` `+0x00`, `item` `+0x04`, `count` `+0x08`, `cars` `+0x0C`):

* **case 0** — `item` is a pointer to a tile `Item` record:
  `pos` `+0x00` (Vector4), `respawn` `+0x14`, `timer` `+0x18`, `info` `+0x1C`,
  `value` `+0x20`, `thing` `+0x24`, `type` `+0x28`, `hasValue` `+0x34` (u8),
  `subtype` `+0x35` (u8), `spawned` `+0x38`.
* **case 1** — `item` is either a **data-pack script ID** (`< 1000`) or a raw
  `ItemInfo *` cast in place.

`ItemInfo`: `type` `+0x00` (`7` = wildcard pickup), `height` `+0x10` (f32),
`effect` `+0x28` (effect ID / ThingInfo ptr).
`ItemParams` (the CreateThing(4) param block, `__attribute__((aligned(16)))`, GC src 663):
`quat` `+0x00` (w `+0x0C`), `pos` `+0x10` (Vector4), three f32 `unk` fields
(GC source names them `unk30/34/38`; PC slots `+0x20/+0x24/+0x28`), `info` `+0x30`,
`value` `+0x34` (f32), `subtype` `+0x38` (u8).

`cars[i]` = TileCar spawn records (`creature` at `+0x28`). Thing fields:
flags `+0x74` (bit 0 = "in world"), tileData backptr `+0x90`, respawn link `+0xD0`.

## Execution

```c
s32 ActionVehicleGiveItem(Action *action) {
    switch (action->type) {
    case 0:
        item = action->item;
        if (item == NULL) {
            return;                        /* BARE return in an s32 function (src 674) */
        }
        if (item->info->type == 7) {       /* wildcard: detach from every player car */
            numPlayers = PlayerManager_PlayerCount();          /* info NOT guarded */
            for (j = 0; j < numPlayers; j++) {
                car = *(u8 **)((u8 *)PlayerManager_GetPlayerByIndex(j) + 0x2C);
                if (car != NULL && *(u8 **)(car + 0x8C) == g_carAVTable) {
                    t = ThingManagerList_GetByType(car + 0x344, 4);
                    if (t != NULL && *(Item **)(t + 0x90) == item) {
                        ThingManagerList_Remove((void **)(car + 0x344), t);
                    }
                }
            }
        }
        if (item->thing == NULL) {                 /* spawn the item thing */
            item->timer = 0;
            params.pos = item->pos;                /* Vector4 copy, 4 dwords */
            params.pos.y += item->info->height;
            params.unk30 = params.unk34 = params.unk38 = 0.0f;
            params.info = item->info;
            params.quat.w = 1.0f;                  /* explicit identity */
            params.quat.x = params.quat.y = params.quat.z = 0.0f;
            params.value   = item->hasValue ? item->value : 0.0f;
            params.subtype = item->subtype;
            thing = ThingManager_DARYL_CreateThing(4, &params);
            if (thing != NULL) {
                item->spawned = 1;
                item->thing = thing;
                thing->tileData = item;            /* +0x90 backptr */
                thing->flags |= 1;                 /* +0x74 */
                if (item->respawn != NULL) {
                    thing->respawn_link = item;    /* +0xD0 */
                }
                if (item->info->effect != NULL) {  /* spawn the pickup effect too */
                    /* p = effect ThingParam block (stack, ~0x60 bytes):
                       a = item->pos (Vector4), b = params.quat, c = {0,0,0},
                       zero = g_zeroVector, id = info->effect,
                       player = 0, f0 = f1 = 0 */
                    ThingManager_DARYL_CreateThing(1, &p);   /* type 1 = effect */
                }
            }
        }
        thing = item->thing;                       /* whatever exists now (src 726) */
        break;
    case 1:
        info = NULL;
        if ((u32)action->item < 1000) {            /* UNSIGNED compare (src 730) */
            packID = FileStreaming_GetPackID(*(char **)(*(u8 **)((u8 *)g_worldData.world + 0x9C) + 0xA4),
                                             2, (u32)action->item);
            if (packID < 0xFFFFFFFF) {
                pack = FileStreaming_InitializeDataPack(packID, 2, 1);
                if (pack != NULL) {
                    info = (ItemInfo *)DataPack_GetScript(pack, (s32)action->item);
                }
            }
        } else {
            info = action->item;                   /* already a raw ItemInfo * */
        }
        if (info != NULL) {
            params2.pos = g_zeroVector;            /* src 742 */
            params2.pos.y += info->height;
            params2.unk30 = params2.unk34 = params2.unk38 = 0.0f;
            params2.info = info;
            params2.quat.w = 1.0f;  x = y = z = 0.0f;   /* identity */
            params2.value = 0.0f;                  /* no value on scripted gives */
            thing = ThingManager_DARYL_CreateThing(4, &params2);
            if (thing != NULL) {
                thing->flags |= 1;
            }
        }
        break;
    default:
        return 0;
    }
    if (thing == NULL) {
        return 0;
    }
    for (i = 0; i < action->count; i++) {          /* hand it to every listed car */
        creature = ((Vehicle *)action->cars[i])->creature;   /* +0x28; vehicle UNguarded */
        if (creature != NULL) {
            CarA_ProcessItemPickup(creature, thing);
        }
    }
    return 1;
}
```

## PC match

`0x00401A90` — `sub esp,0x8c` then `push ebx/ebp/esi/edi`, `mov edi,[esp+0xa0]` = action.
The `0x8C` frame is the fingerprint: it exists only to 16-byte-align `params` at
`[esp+0x10]` (GC `aligned(16)`, src 663).

* Dispatch: `mov eax,[edi]; sub eax,ebp; je 0x00401BAB` (case 0) / `dec eax;
  jne 0x00401D9A` (default → `xor al,al` = `return 0`).
* **NULL item**: `mov esi,[edi+4]; cmp esi,ebp; je 0x00401DD8` @`0x00401BB0` — jumps
  into the epilogue **past** `mov al,bl` (`0x00401DD6`), so the return value is whatever
  is in `al`: `ebx` is callee-saved (pushed at entry) and its `=1` staging only happens
  at `0x00401ABC` (case 1) / `0x00401C22` (case 0 spawn guard) — **uninitialized
  garbage return**, the exact compiled form of GC's bare `return;` (src 674).
* Wildcard block: `mov eax,[esi+0x1c]; cmp byte ptr [eax],7` @`0x00401BB9` — `info`
  deref **unguarded** ✓ GC 676. `call 0x00440160` (PlayerCount) + `and eax,0xff`
  (count zero-extended from a byte), `mov ebx,eax; jbe 0x00401C1B` skips the loop.
* Player loop @`0x00401BCC`: `call 0x00440180` (GetPlayerByIndex), `mov eax,[eax+0x2c]`
  (player→car), guard, `cmp dword ptr [eax+0x8c],0x47a2e8` (**CarA vtable address as
  immediate**), `lea edi,[eax+0x324]` (PC thing-list offset), `push 4; push edi;
  call 0x0044A840` (GetByType), `cmp [eax+0x90],esi` (backptr == item),
  `push eax; push edi; call 0x0044A7C0` (List_Remove); `mov edi,[esp+0xa0]` reloads
  action (edi reused as list pointer), `inc ebp; cmp ebp,ebx; jb` (re-reads nothing —
  count live in ebx).
* Spawn guard @`0x00401C1B`: `mov eax,[esi+0x24]; cmp eax,ebp; mov ebx,1;
  jne 0x00401D93` ✓ GC 688 (already-spawned skips to `thing = item->thing`).
* `params` fill @`0x00401C2D`–`0x00401CA1` (base `[esp+0x10]`): 4-dword pos copy from
  `[esi+0]`, height via `fld [esp+0x24]; fadd dword ptr [eax+0x10]; fstp [esp+0x24]`,
  `mov [esi+0x18],ebp` (timer=0), `mov [esp+0x40],eax` (info), zero `fld`s
  (`[esp+0x30/34/38],ebp`), quat as **pure immediates** `mov [esp+0x1C],0x3f800000` +
  3× `mov …,0`, `item->hasValue ? item->value : 0.0f` as a branch pair writing
  `[esp+0x44]`, subtype `mov dl,[esi+0x35]` → `mov byte ptr [esp+0x50],dl` (slot
  recomputed across the `push eax; push 4` args) → `call 0x0044D0B0` (CreateThing)
  @`0x00401CA5`.
* Success @`0x00401CB5`: `mov [esi+0x38],bl` (spawned, **full dword**),
  `mov [esi+0x24],eax`, `mov [eax+0x90],esi`, `or [eax+0x74],ebx`; respawn
  `cmp [esi+0x14],ebp / mov [eax+0xd0],esi`; effect guard
  `mov eax,[esi+0x1c]; mov ecx,[eax+0x28]; je 0x00401D93`; p block @`0x00401CDD`–
  `0x00401D84`: pos 4 dwords, quat copied from `params` (`[esp+0x10..0x1C]` →
  `[esp+0x5C..0x68]`), three 0s, **g_zeroVector from globals `[0x00692350]`/
  `[0x00692354]`/`[0x00692358]`/`[0x0069235C]`**, id=effect, player/f0/f1=0;
  `push ecx` (&p); `push ebx` (**type 1** — reuses the staged 1);
  `call 0x0044D0B0` @`0x00401D8B`.
* `0x00401D93: mov esi,[esi+0x24]` = GC 726 `thing = item->thing` → shared tail
  `cmp esi,ebp` @`0x00401D96`; case 1 joins at `0x00401D96` via `jmp` @`0x00401BA6`.
* Case 1 @`0x00401AB4`: `mov eax,[edi+4]; cmp eax,0x3e8; mov ebx,1; jae 0x00401B0B` —
  `id >= 1000` falls to the info guard with **`eax` used directly as `info`** ✓ GC 739.
  Pack chain `mov ecx,[0x0049DBA0]; mov edx,[ecx+0x9c]; mov ecx,[edx+0xa4]` +
  `push id; push 2; push ecx; call 0x004289E0` (GetPackID); `cmp eax,-1; jae
  0x00401D9A`; `push ebx (=1); push 2; push eax; call 0x00428C10`
  (InitializeDataPack); pack guard; `call 0x004216C0` (GetScript); `je 0x00401D9A` —
  all three failure arms return 0 directly (same visible result as GC's tail check,
  since `thing` would stay NULL). `params2` @`0x00401B13`: pos from the g_zeroVector
  globals with `fld [esp+0x24]; fadd dword ptr [eax+0x10]` (height; the `fstp` operand
  is **recomputed after** `push edx` — same slot either way), zeros, explicit quat,
  value 0; `call 0x0044D0B0` @`0x00401B91`; `or [esi+0x74],ebx`.
* Tail @`0x00401DA7`: `mov eax,[edi+8]` (count), `jbe 0x00401DD6`; `add edi,0xc`;
  loop `mov eax,[edi]; mov eax,[eax+0x28]` (vehicle **not** guarded, creature guarded),
  `push esi; push eax; call 0x00412270` (CarA_ProcessItemPickup); count re-read from
  `[[esp+0xa0]+8]` every iteration; `mov al,bl` @`0x00401DD6` = `return 1`; shared
  epilogue `pop edi/esi/ebp/ebx; add esp,0x8c; ret` @`0x00401DD8`.

## Version differences

* **Garbage-return quirk**: GC src 674 is a bare `return;` inside an `s32` function.
  MSVC compiles it to entering the epilogue past the `return 1` write — `je 0x00401DD8`
  leaves `al` = caller's `ebx` low byte (`0x00401DD6` never executed). Any "cleaned up"
  reconstruction (`return 0;`) would silently change the PC binary's contract with the
  action dispatcher; keep the quirk explicit.
* **Aligned param block**: GC `__attribute__((aligned(16)))` (src 663) is what forces
  MSVC's `sub esp,0x8c` with `params` at `[esp+0x10]` — the unique frame size of the
  whole unit, and the fastest way to spot this function in a raw dump. PS2 keeps
  `params` in a `0x74`-relative stack slot (`add.s` height target, `0x0024E3A0`).
* **Constant materialization**: GC writes the identity quaternion and zeros as explicit
  component assignments (src 695–698, 742–751); PC emits them as **immediate stores**
  (`mov [esp+0x1C],0x3f800000`) and reads g_zeroVector from globals
  `0x00692350`–`0x0069235C`; PS2 materializes `1.0f` inline (`lui $2,0x3F80`+`mtc1`
  @`0x0024E370`) but **block-copies the constant quaternion scalar-wise** from .rodata
  `D_2DEED0` (`0x0024E440` effect block, `0x0024E544` params2). Three builds, three
  constant idioms — match on fields, not on instructions.
* **Unsigned ID compare**: `(u32)action->item < 1000` — PC `cmp eax,0x3e8; jae`
  @`0x00401AB7`/`0x00401AC1`, PS2 `sltiu $x,$y,0x3E8` @`0x0024E4E4`. A signed compare
  reconstruction (`< 1000` on `s32`) would diverge for IDs ≥ 0x80000000, which are
  already raw pointers; keep the cast.
* **Thing-list skew**: `car + 0x344` (GC src 681/683) vs PC `lea edi,[eax+0x324]`
  @`0x00401BE8` — the known CarA list offset difference (also in
  `ActionCreatureChangeMode` / `ActionVehicleChangeMode`).
* **Delay-slot staging on PS2**: the effect block's `a` Vector4 copy straddles the
  `jal ThingManager_DARYL_CreateThing` @`0x0024E4D0` with its last float in the delay
  slot — `swc1 $f0,0xC($2)` @`0x0024E4D4`; `AddItemPack`'s third arg is staged in the
  InitializeDataPack delay slot (`addiu $6,$0,1` @`0x0024E518`). PC has no delay slots
  (but does recompute push-shifted stack offsets, e.g. `mov [esp+0x50],dl` @`0x00401CA1`).
* **Vtable identity test**: GC `== g_carAVTable` extern; PC compares against the raw
  vtable address **as an immediate** (`cmp [eax+0x8c],0x47a2e8`); PS2 loads it via
  `lui %hi(D_002C5120)` @`0x0024E310`. The immediate/global is a reliable PC↔PS2 link.
* **Call-graph legibility**: PS2 has named `jal`s (PlayerCount `0x0024E2E0`,
  GetPlayerByIndex `0x0024E2F8`, GetByType `0x0024E320`, List_Remove `0x0024E33C`,
  GetPackID `0x0024E500`, GetScript `0x0024E52C`, two CreateThing calls
  `0x0024E3F0`/`0x0024E4D0`); PC is stripped, so use the constant/shape set above
  (0x3E8 compare, `0x3f800000`, `0x47a2e8`, pack-chain pointer chase
  `[[0x49DBA0]+0x9C]+0xA4`) as the matching keys.
* **Case-1 failure arms**: PS2 and PC short-circuit packID/pack/info failures straight
  to the `return 0` epilogue; GC source lets `info` stay NULL and falls through the
  tail (same observable result since `thing` stays NULL). Behaviorally identical —
  don't chase it as a difference.
