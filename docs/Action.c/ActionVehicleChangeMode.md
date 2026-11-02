# `ActionVehicleChangeMode`

## Signature

* **GC** `s32 ActionVehicleChangeMode(u32 *action)` — `0x80006030`, size `0x3BC`
  (`HWVX-GC-Decomp/src/Action.c:308`).
* **PS2** `s32 ActionVehicleChangeMode(s32 *action)` — `0x0024E920`, size `0x45C`
  (`asm/nonmatchings/cod/ActionVehicleChangeMode.s`; no m2c draft).
* **PC** `s32 __cdecl ActionVehicleChangeMode(u32 *action)` — `0x00401550`, size `0x370`
  (unit `text_00401090`; extent `0x00401550`–`0x004018BF`, jump table `0x0040189C` + case-value
  map `0x004018B0`). **PC-CONFIRMED** (full disassembly verified).

## Arguments

`action` = record's `data`: `action[0]` = mode, `action[1]` = list length,
`action[2 + i]` = **TileCar** spawn record pointer. TileCar layout (from the GC typedef):
pos `+0x00`, rotX/Y/Z `+0x10/+0x14/+0x18`, aiScripts `+0x24`, creature `+0x28`,
mode `+0x2C`, `unk34` `+0x34`, nearDist/farDist `+0x38/+0x3C`, `scripts[4]` `+0x40..0x4C`,
item `+0x54`.

Modes: `0` despawn, `1`/`10` spawn AI, `3` end-event + release, `9` set player flag.

## Execution

```c
s32 ActionVehicleChangeMode(u32 *action) {
    /* typedefs: TileCar (above); CreatureParams {pos@0x00, quat@0x10, vel@0x20,
       aiScripts@0x30, player@0x3C}; AIPlayerDesc {script@0x00, nearSq@0x04,
       farSq@0x08, unkC@0x0C, unkD@0x0D}; AIPlayer {aiScripts@0x3C, flags@0x6C,
       tileCar@0xA8} */
    mode  = action[0];
    count = action[1];
    for (i = 0; i < count; i++) {
        spawn    = (TileCar *)action[i + 2];
        creature = spawn->creature;
        player   = (AIPlayer *)Action_GetCarPlayer(creature);
        switch (mode) {
        case 0:                                       /* despawn */
            if (creature != NULL) {
                paused = Trigger_IsTACSystemPaused();
                Trigger_PauseTACSystem(1);
                spawn->pos = creature->worldMatrix.v[3];   /* +0x40, 16-byte copy */
                CarA_Release(creature);
                PlayerManager_RemovePlayer(player);
                spawn->creature = NULL;
                Trigger_PauseTACSystem(paused);            /* restore, not force */
            }
            break;
        case 9:
            player->flags |= 1;                      /* UNGUARDED player */
            break;
        case 3:
            if (creature != NULL) {
                Thing_ProcessEndEvent(creature, creature->vehicleIface->collider, 0, 0);
                CarA_Release(creature);
            }
            break;
        case 1:
        case 10:                                      /* spawn AI driver */
            if (creature == NULL && spawn->unk34 == 0) {
                difficulty = GameSettings_GetDifficulty();
                params.pos = spawn->pos;  params.pos.y += 10.0f;
                params.aiScripts = spawn->aiScripts;  params.vel = g_zeroVector;
                SetOrientation(&mtx, spawn->rotY, spawn->rotX, spawn->rotZ);
                SubMatrixToQuaternion(&mtx, &params.quat);
                /* script select: spawn->scripts[difficulty] preferred; else
                   params.aiScripts->scripts[difficulty] BUT each tier's fallback
                   is guarded by scripts[0] != 0 (quirk), and difficulty>3 uses
                   the chain scripts[0]->[1]->[2]; else script stays 0 */
                desc.script = script;  desc.unkD = 0;  desc.unkC = spawn->unk34;
                desc.nearSq = spawn->nearDist * spawn->nearDist;
                desc.farSq  = spawn->farDist  * spawn->farDist;
                ai = PlayerManager_NewAIPlayer(&desc);
                if (ai != NULL) {
                    params.player = ai;
                    spawn->creature = ThingManager_DARYL_CreateThing(9, &params);
                    if (spawn->creature != NULL) {
                        spawn->creature->tileData = spawn;          /* thing +0x90 */
                        Player_SetThing(ai, spawn->creature);
                        Player_SetName(ai, -1, "AI");
                        ai->tileCar   = spawn;
                        ai->aiScripts = params.aiScripts;
                        if (spawn->item != NULL) {          /* stock the wildcard rack */
                            rack = ThingManagerList_GetByType(
                                       &spawn->creature->motionState.slaves, 8);
                            WildCardRack_JettisonAll(rack);
                            WildCardRack_Add(rack, spawn->item,
                                             spawn->item->params->count);
                        }
                    } else {
                        PlayerManager_RemovePlayer(ai);             /* undo AI player */
                    }
                }
            }
            if (player != NULL) {
                player->flags &= ~1;                 /* guarded, opposite of case 9 */
            }
            break;
        }
        spawn->mode = mode;                          /* every mode, every spawn */
    }
    return 1;
}
```

## PC match

`0x00401550` — mode dispatch is MSVC's remapped jump table: case-value map at `0x004018B0`
compresses `{0,1,3,9,10}` to indices, table at `0x0040189C`.

* Case 0: `call 0x0044F210` (IsTACPaused) / `call 0x0044F200` (PauseTAC, `1` then the saved
  flag — the **restore**, not force), 16-byte pos copy from `[creature+0x40]`
  (worldMatrix.v[3]), `call 0x0040DAB0` (CarA_Release), `call 0x00440360` (RemovePlayer),
  `mov [spawn+0x28],0` ✓ GC 408.
* Case 9: `or dword ptr [player+0x6C],1` **unguarded** ✓ GC 413 quirk.
* Case 3: `call 0x0044A430` (Thing_ProcessEndEvent) with collider via
  `mov eax,[creature+0x398]; mov eax,[eax+0xA0]` ✓ GC 417 (`vehicleIface` +0x398,
  `collider` +0xA0).
* Case 1/10: guards `test creature,creature` / `[spawn+0x34]` == 0; `fadd dword ptr
  [0x00477004]` (**10.0f as a .rdata global**); script cascade reads `[esi+0x40..0x4C]`
  (TileCar scripts) with fallback `[aiScripts+0x90..0x9C]` and the **scripts[0]-guard quirk**
  verbatim @`0x00401707`+; `call 0x004400C0` (NewAIPlayer) @`0x004017B9`;
  `call 0x0044D0B0` (CreateThing, `9`) @`0x004017D6`; `call 0x0043EC10` (SetThing);
  `call 0x0043EAC0` (SetName, `-1`, `"AI"` @`0x0047A164`); `ai+0xA8 = spawn`,
  `ai+0x3C = aiScripts`; rack section `call 0x0044A840` (GetByType, list `[car+0x324]`,
  type 8) + `call 0x00451C00` (JettisonAll) + `call 0x00451C80` (Rack_Add, count from
  `[item+0x30]`→`+0x20`); CreateThing failure → RemovePlayer; tail
  `and dword ptr [player+0x6C],~1` guarded ✓ GC 488–490; `[spawn+0x2C] = mode` ✓ GC 493.
* PC call targets: SetOrientation `0x00453B20`, SubMatrixToQuaternion `0x004540C0`,
  GameSettings_GetDifficulty `0x0042AF90`.

## Version differences

* **Thing list offset**: GC/PS2 use creature `+0x344` for the slaves list (PS2
  `addiu $4,$2,0x344` in GetByType's **delay slot** @`0x0024ECD`); PC uses `+0x324` — the
  CarA/Thing layout differs on PC (same skew seen in `ActionCreatureChangeMode`).
* **10.0f constant**: GC/PC read it from a data global (`[0x00477004]` on PC); PS2
  materializes it inline with `lui $2,0x4120` + `mtc1` @`0x0024EA98` (no .rodata float pool
  entries for these constants in the SN build) — a strong PS2 fingerprint.
* **Mode dispatch shape**: GC compiles the `switch` into a branch cascade; PS2 stages mode
  constants in registers and cascades `beq` in order **10,1 (merged @`.L0024EA70`), 3
  (@0x0024E9A8), 9 (@0x0024E9B0), beqz 0 (@0x0024E9B8)**; PC uses a case-value-map + jump
  table (`0x004018B0`/`0x0040189C`). All preserve the 1/10 merged body.
* **Case 1/10 guards**: PS2 implements `creature == NULL && spawn->unk34 == 0` as
  `bnez $18,.L0024ED18` / `bnez $2,.L0024ED18` @`0x0024EA70/7C` — both fall through to the
  **guarded flags-clear tail** (so skipped spawn still clears bit 0) ✓ GC 488–490; PC the
  same via two tests jumping past the body.
* **Delay-slot staging on PS2** (no PC/GC equivalent): `PlayerManager_RemovePlayer` arg
  staged in a delay slot @`0x0024ED14`; `Player_SetName`'s `"AI"` pointer
  (`%lo(D_002CDAB8)`) staged in SetName's delay slot @`0x0024ECB8` (`-1` @`0x0024ECB0`);
  `spawn->mode = mode` (`sw $16,0x2C($17)`) sits in the **loop-back branch's delay slot**
  @`0x0024ED38`; `Player_SetThing` arg in its delay slot (`daddu $4,$18,$0` @`0x0024ECA4`).
* **Script cascade quirk in all three builds**: each difficulty tier's `aiScripts->scripts[k]`
  fallback is guarded by **`scripts[0] != 0`**, not `scripts[k]` (GC 435/441/447/453; PC
  @`0x00401707`+; PS2 cascade @`0x0024EB28`–`0x0024EC08`) — an original bug preserved
  identically; do not "fix" it.
* PS2 keeps `action` spilled at `0xAC($29)` in a `0x140`-byte frame (vs PC tight register use);
  `creature->tileData = spawn` is `sw $17,0x90($2)` @`0x0024EC98` right after CreateThing's
  return, matching PC `[thing+0x90]` store.
