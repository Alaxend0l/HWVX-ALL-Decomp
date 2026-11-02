# `ThingManagerList_TriggerSlaves`

Interprets one `SlaveTrigger` record against a thing list: switches on
`trigger->type`, builds a type-specific parameter block on the stack, spawns a
slave through `ThingManager_DARYL_CreateSlave`, stamps it with
`slavePlayer = player`, and prepends it onto `*list` (the thing's private
child list — the slave stays out of the manager `activeList` until
`ThingManagerList_EmancipateSlaves` promotes it). Two cases spawn nothing:
type `10` initialises a shared-slave holder record via
`SharedSlaveHolder_Get`, and type `99` releases every slave in `*list`
matching `slaveId`/`slavePlayer`. This is how motion-state trigger tables
materialise event sequences, emitters, attaches, tracers, wild-card racks,
motions, sound sequences and radar blips as slaves — see
`ThingManagerList_InitAllSlavesNow` for the loop that feeds it.

## Signature

* **GC** `void ThingManagerList_TriggerSlaves(ThingHeader **list, struct SlaveTrigger *trigger, Vector4 *a, Vector4 *b, Vector4 *c, u32 player)`
  — `0x80054964`, size `0x6B0` (`HWVX-GC-Decomp/src/ThingManager.c:350`).
* **PS2** — `0x002B04C0`, size `0xA94`
  (`asm/nonmatchings/cod/ThingManagerList_TriggerSlaves.s`, 752 lines).
* **PC** — `0x0044AD00`, size `0x610`, in unit `text_0044AD00`
  (`config/hotwheels/functions.json`).

## Arguments

* `list` — `ThingHeader **`, destination slave chain (a thing's private child
  list; the external caller clones pass `(state + 0x14)`-style lists).
* `trigger` — `SlaveTrigger *`, laid out per `ThingManager.c:350`–`:357`:

  ```c
  struct SlaveTrigger {   /* size 0x20 */
      Vector4 pos;        /* 0x00 */
      u8      type;       /* 0x10 */
      u8      pad11;      /* 0x11 */
      u16     id;         /* 0x12 */
      u8      pad14[4];   /* 0x14 */
      void   *data;       /* 0x18 */
  };
  ```

* `a`, `b`, `c` — `Vector4 *` forwarded verbatim into the parameter blocks
  (`a` is the spawn position; `ThingManagerList_InitAllSlavesNow` passes the
  trigger record itself as `a`, a `{0,0,0,1}` direction as `b`, and
  `g_zeroVector` as `c`).
* `player` — declared `u32` by m2c but only ever byte-traffic'd: compared to
  the `slavePlayer` byte (`0x83`) in case 99 and stored into `0x83` in the
  tail.

## Execution

Case-to-pool map (GC source literals cross-checked against the PS2 inlined
freeList slots and the PC `push <type>` immediates):

| type | pool spawned | guard | `.player = g_pSourcePlayer` | notes |
|-----:|:-------------|:------|:---:|:------|
| 1 | 1 `eventSeqs` | `data != NULL && *(u32 *)data != 0` | ✓ (written twice, `:387`/`:390`) | `p1.zero = g_zeroVector`, `p1.f = 0.0f` |
| 2 | 2 `emitters` | `data != NULL` | ✓ | `p2.f = 0.0f`, `p2.a.w = 1.0f` |
| 3 | 0x10 `soundSeqs` | `data != NULL` (duplicated — m2c artifact, `:411`) | — | `p3.flag = 1` |
| 4 | 8 `wildCardRacks` | none | — | sole param `p4 = trigger->pos`, `p4.w = 1.0f` |
| 5 | 5 `attaches` | `data != NULL` | ✓ | `p5.pos = trigger->pos`, `p5.flag = 1` |
| 6 | 5 `attaches` | `ViewManager_GetViewFormat() > 1 && *(u8 *)trigger->data != 0 → break`, then `data != NULL` | ✓ | `p6.pos = trigger->pos`, `p6.flag = 0`; data is **dereferenced** at `:433` before its NULL check at `:436` |
| 7 | 0xC `tracers` | `data != NULL` | ✓ | `p7.time = 20`, `p7.f = 0.0f`, `p7.a.w = 1.0f` |
| 8 | 0xF `motions` | `data != NULL` | — | `p8.flag = 0` |
| 9 | 0x11 `radarBlips` | `data != NULL` | — | `p9 = {a, data}` |
| 10 | — | `data != NULL && ((SharedSlaves *)data)->active == 0` | — | `active = 1`, `info[0x1D] = SharedSlaveHolder_Get(shared)` (`:475`–`:481`) |
| 99 | — | — | — | walk `*list` via `next` (`0xA4`), `slaveId == trigger->id && slavePlayer == player → vtable[1](thing)` (`:482`–`:490`) |
| default | none | — | — | — |

Reconstruction (GC `:383`–`:501`; parameter struct shapes per `:367`–`:375`):

```c
switch (trigger->type) {
case 1:
    p1.data = trigger->data;
    p1.c = *c;
    p1.player = g_pSourcePlayer;
    p1.zero = g_zeroVector;
    if (p1.data != NULL && *(u32 *)p1.data != 0) {
        p1.player = g_pSourcePlayer;      /* second write — m2c ordering artifact */
        p1.f = 0.0f;
        p1.a = *a; p1.a.w = 1.0f;
        p1.b = *b;
        slave = ThingManager_DARYL_CreateSlave(1, &p1, trigger->id);
    }
    break;
case 4:                                   /* no data guard */
    p4 = trigger->pos;
    p4.w = 1.0f;
    slave = ThingManager_DARYL_CreateSlave(8, &p4, trigger->id);
    break;
case 6:
    if (ViewManager_GetViewFormat() > 1 && *(u8 *)trigger->data != 0) {
        break;                            /* region-dependent attach */
    }
    if (trigger->data != NULL) {
        p6.data = trigger->data;
        p6.player = g_pSourcePlayer;
        p6.pos = trigger->pos;
        p6.flag = 0;
        slave = ThingManager_DARYL_CreateSlave(5, &p6, trigger->id);
    }
    break;
case 10:
    shared = trigger->data;               /* SharedSlaves record */
    if (shared != NULL && shared->active == 0) {
        shared->active = 1;
        shared->info[0x1D] = SharedSlaveHolder_Get(shared);
    }
    break;
case 99:
    for (thing = *list; thing != NULL; thing = thing->next) {
        if (thing->slaveId == trigger->id && thing->slavePlayer == player) {
            thing->vtable[1](thing);      /* Release */
        }
    }
    break;
/* cases 2, 3, 5, 7, 8, 9 are the same data-guarded CreateSlave shape */
}

if (slave != NULL) {                      /* shared tail, :493-:501 */
    slave->slavePlayer = player;
    slave->next = *list;
    slave->prev = NULL;
    if (slave->next != NULL) {
        slave->next->prev = slave;
    }
    *list = slave;                        /* head-insert */
}
```

Note the reinterpretation the case-10 tail relies on: the m2c
`SharedSlaves { u32 active; u8 pad4[4]; u8 *info; }` (`:358`–`:362`) is the
same record `ThingManagerList_InitAllSlavesNow` takes as
`(u8 *)shared + 4` — `pad4[4]` is the trigger **count** and `info` the trigger
array base (`SlaveList { u32 count; SlaveTrigger *triggers; }`). The
holder-index scratch byte lands at `triggers[0] + 0x1D`.

## PC match

`0x0044AD00` (size `0x610`) carries 26 relocs; the identifying set:

* `[34, 6, None, 1452]` — byte-LUT displacement → table at `0x0044B2AC`;
* `[41, 6, None, 1404]` — jump-table displacement → table at `0x0044B27C`;
* `[91, 6, '_D_00692350']`, `[104, '_D_00692354']`, `[117, '_D_00692358']`,
  `[143, '_D_0069235C']` — the four `g_zeroVector` components (case 1
  `p1.zero`);
* `[130, '_D_00489528']`, `[380]`, `[684]`, `[790]`, `[915]` — exactly five
  `g_pSourcePlayer` loads, i.e. cases 1, 2, 5, 6, 7;
* `[713, 20, '_func_0044D0F0']` — one shared `CreateSlave` call site
  (`0x0044AFC8`);
* `[747, 20, '_func_0044A790']` — `ThingManagerList_Append` in the tail;
* `[764, 20, '_func_00450A50']` — `ViewManager_GetViewFormat` at the case-6
  label;
* `[1307, 20, '_func_0044AAB0']` — `SharedSlaveHolder_Get` in case 10;
* twelve jump-table dwords at `+1404`…`+1448` with extras
  `45, 314, 520, 589, 641, 763, 853, 1072, 1212, 1279, 1329, 754` — targets
  `0x0044AD2D, 0x0044AE3A, 0x0044AF08, 0x0044AF4D, 0x0044AF81, 0x0044AFFB,
  0x0044B055, 0x0044B130, 0x0044B1BC, 0x0044B1FF, 0x0044B231, 0x0044AFF2` =
  cases 1–10, 99, default.

The byte LUT (99 bytes at `0x0044B2AC`, indexed `type - 1`) reads
`00 01 02 03 04 05 06 07 08 09` then `0b` ×88 then `0a` — types 1–10 map to
jump slots 0–9, type 99 to slot 10, every other type to slot 11 (the
default).

Dispatch head — argument slots after `sub esp,0x178` + `push ebx` +
`push esi` are `list=[esp+0x184]`, `trigger=[esp+0x188]`, `a=+0x18C`,
`b=+0x190`, `c=+0x194`, `player=+0x198`:

```asm
0044AD00: sub   esp, 0x178              ; locals incl. p1..p9
0044AD0A: mov   esi, [esp+0x188]        ; trigger
0044AD11: mov   al, [esi+0x10]          ; trigger->type
0044AD14: dec   eax
0044AD15: cmp   eax, 0x62               ; 1..99 window
0044AD18: ja    0x44aff2                ; out of range -> shared epilogue
0044AD20: mov   cl, [eax+0x44b2ac]      ; byte LUT
0044AD26: jmp   dword [ecx*4+0x44b27c]  ; jump table
```

Case 1 (`0x0044AD2D`) fills `p1` at `[esp+0xF8]` — field order matches GC
`:367` exactly: `a@0xF8, b@0x108, c@0x118, zero@0x128, data@0x138,
player@0x13C, f@0x140`:

```asm
0044AD36: mov   eax, [esi+0x18]         ; trigger->data
0044AD39: test  eax, eax                ; guard part 1 ...
0044AD3B: mov   [esp+0x118], ecx        ;   ... struct fill hoisted between
0044AD59: mov   edx, [0x692350]         ; g_zeroVector.x -> p1.zero
0044AD80: mov   ecx, [0x489528]         ; g_pSourcePlayer
0044AD93: mov   [esp+0x138], eax        ; p1.data
0044AD9A: mov   [esp+0x13C], ecx        ; p1.player  (write #1, GC :387)
0044ADA8: je    0x44aff2                ;   deferred data != NULL branch
0044ADAE: cmp   dword [eax], 0          ; *(u32 *)data != 0 (GC :389)
0044ADB1: je    0x44aff2
0044ADBE: mov   [esp+0x13C], ecx        ; p1.player again (write #2, GC :390)
0044AE0C: xor   eax, eax
0044AE0E: mov   al, [esi+0x12]          ; trigger->id (low byte only)
0044AE18: lea   ecx, [esp+0xf8]         ; &p1
0044AE1F: mov   dword [esp+0x140], 0    ; p1.f = 0.0f
0044AE31: push  eax                     ; id
0044AE32: push  ecx                     ; &p1
0044AE33: push  1                       ; pool 1
0044AE35: jmp   0x44afc8                ; shared CreateSlave call site
```

MSVC merges the spawning cases onto one call site at `0x0044AFC8`: cases 5
and 6 fall through the `push 5` at `0x0044AFC6`, case 8 converges with
`push 0xf / jmp 0x44afc8` (`0x0044B1B5`), case 9 with
`push 0x11 / jmp 0x44afc8` (`0x0044B1F8`) — matching the GC pool literals
exactly. The shared tail reproduces GC `:493`–`:501` with an
`ThingManagerList_Append` call instead of an inlined splice, then the single
shared epilogue:

```asm
0044AFC6: push  5                       ; type (cases 5/6; others push own)
0044AFC8: call  0x44d0f0                ; ThingManager_DARYL_CreateSlave
0044AFCD: add   esp, 0xc
0044AFD0: test  eax, eax
0044AFD2: je    0x44aff2                ; slave == NULL
0044AFD4: mov   dl, [esp+0x198]         ; player arg
0044AFDB: push  eax                     ; slave
0044AFDC: mov   byte [eax+0x83], dl     ; slave->slavePlayer = player
0044AFE2: mov   eax, [esp+0x188]        ; list
0044AFE9: push  eax
0044AFEA: call  0x44a790                ; ThingManagerList_Append(list, slave)
0044AFEF: add   esp, 8
0044AFF2: pop   esi                     ; shared epilogue (also default target)
0044AFF3: pop   ebx
0044AFF4: add   esp, 0x178
0044AFFA: ret
```

Case 6 (`0x0044AFFB`) opens with the view-format gate, mirroring the GC
short-circuit — and the same dereference-before-NULL-check ordering
(`:433` deref vs `:436` check):

```asm
0044AFFB: call  0x450a50                ; ViewManager_GetViewFormat()
0044B000: cmp   eax, 1                  ; <= 1 skips the deref ...
0044B003: jbe   0x44b00d
0044B005: mov   ecx, [esi+0x18]
0044B008: cmp   byte [ecx], 0           ; *(u8 *)trigger->data != 0 ...
0044B00B: jne   0x44aff2                ; -> break
0044B00D: mov   eax, [esi+0x18]
0044B010: test  eax, eax                ; data NULL check AFTER the deref
```

Case 10 (`0x0044B1FF`) calls the confirmed `SharedSlaveHolder_Get` and writes
the holder-index scratch byte at `triggers[0] + 0x1D` (GC `:479`), then exits
through its own copy of the epilogue:

```asm
0044B1FF: mov   esi, [esi+0x18]         ; shared = trigger->data
0044B202: test  esi, esi
0044B204: je    0x44aff2
0044B20A: cmp   dword [esi], 0          ; shared->active == 0
0044B20D: jne   0x44aff2
0044B213: push  esi
0044B214: mov   dword [esi], 1          ; shared->active = 1
0044B21A: call  0x44aab0                ; SharedSlaveHolder_Get(shared)
0044B21F: mov   ecx, [esi+8]            ; sl->triggers (= info)
0044B222: add   esp, 4                  ; Get arg cleanup
0044B225: pop   esi                     ; epilogue folded in early (cold block)
0044B226: mov   byte [ecx+0x1d], al     ; info[0x1D] = index (GC :479)
0044B229: pop   ebx
0044B22A: add   esp, 0x178
0044B230: ret
```

Case 99 (`0x0044B231`) walks the list without a next-prefetch — the GC loop
is `thing = thing->next` (`:485`) and vtable[1] may unlink the node, which
both platforms accept:

```asm
0044B231: mov   edx, [esp+0x184]        ; list
0044B238: mov   bl, [esi+0x12]          ; trigger->id (low byte)
0044B23B: mov   esi, [edx]              ; thing = *list
0044B23F: je    0x44aff2
0044B245: cmp   byte [esi+0x82], bl     ; thing->slaveId
0044B24B: jne   0x44b269
0044B24D: mov   al, [esp+0x198]         ; player
0044B254: cmp   byte [esi+0x83], al     ; thing->slavePlayer
0044B25A: jne   0x44b269
0044B25C: mov   ecx, [esi+0x8c]         ; vtable
0044B263: call  dword [ecx+4]           ; vtable[1] = Release(thing)
0044B269: mov   esi, [esi+0xa4]         ; thing = thing->next
0044B271: jne   0x44b245                ; loop
0044B273: pop   esi                     ; second duplicated epilogue
```

The PS2 counterpart pins the same shape from symbols
(`HWVX-PS2-Decomp/symbol_addrs.txt`): a flat `beq` chain
(`0x002B04F0`–`0x002B055C`, comparing in order 99, 10, …, 1; the
`$16 = 0` init sits in the first delay slot at `0x002B04F4`, default branches
to the tail `.L002B0F04` at `0x002B055C`), `jal ViewManager_GetViewFormat`
at `0x002B0A38` inside case 6 (gate: `sltiu $1,$2,2` / `bnez` / `lbu 0(data)`
/ `bnez .L002B0F04` at `0x002B0A38`–`0x002B0A58`), and the identical five
`g_pSourcePlayer` loads (`lw %gp_rel(...)`, abs `0x002D01F8`) at
`0x002B05A4`, `0x002B071C`, `0x002B09B8`, `0x002B0A68`, `0x002B0B8C` —
exactly cases 1, 2, 5, 6, 7, the same five reloc sites as PC
(`+130/+380/+684/+790/+915`). PS2 frame size is `0x220`
(`addiu $29,$29,0x220` in the epilogue @`0x002B0F50`).

## Version differences

* **Switch lowering.** PC uses MSVC's two-level dispatch (99-byte LUT +
  12-entry jump table, `0x0044AD00`–`0x0044AD26`); GC is a plain m2c
  `switch`; PS2 (SN-R5900) has no table at all — a linear `beq` chain
  (`0x002B04F0`–`0x002B055C`) comparing `type` in descending case order
  (99 first, 1 last).
* **`CreateSlave` inlining.** PS2 inlines `ThingManager_DARYL_CreateSlave`
  per case — direct freeList pop from each pool slot (`lui` of
  `0x66A030 + slot`: case 1 `+0x7AB78`, case 2 `+0x7AB7C`, case 3 `+0x7ABB4`,
  case 4 `+0x7AB94`, cases 5/6 `+0x7AB88`, case 7 `+0x7ABA4`, case 8
  `+0x7ABB0`, case 9 `+0x7ABB8`) plus an open-coded `vtable[0]` init
  (`jalr $2` at `0x002B0698`, `0x002B07B4`, `0x002B0884`, `0x002B093C`,
  `0x002B0A0C`, params pointer in the delay slot). GC and PC both **call**
  the helper; MSVC converges every spawning case onto the single call site
  at `0x0044AFC8` via `jmp`/fall-through (shared `push 5` at `0x0044AFC6`
  for cases 5+6; `push 0xf / jmp` @`0x0044B1B5`; `push 0x11 / jmp`
  @`0x0044B1F8`). The per-case pool number is therefore visible as an
  immediate on PC, as a freeList slot offset on PS2, and as the C literal on
  GC — all three agree with the pool table.
* **Tail splice.** GC (`:493`–`:501`) and PS2 (tail @`0x002B0F08`–`0x002B0F30`:
  `beqz $16`, `sb $17,0x83($16)`, inline head-insert) write the 4-store
  head-insert inline; PC calls `ThingManagerList_Append` (`0x0044A790`) at
  `0x0044AFEA` — see that doc for the identical store order.
* **Case 1 artifacts mirror across platforms.** The double
  `p1.player = g_pSourcePlayer` (GC `:387` + `:390`) appears as two
  `mov [esp+0x13C], ecx` stores on PC (`0x0044AD9A`, `0x0044ADBE`) and both
  guard halves branch on all platforms (PC defers the first `je` to
  `0x0044ADA8` after hoisting the struct fill; PS2 branches twice at
  `0x002B05D0`/`0x002B05E0`).
* **Case 3's duplicated NULL test** (GC `:411`, `data != NULL && data !=
  NULL`) is likewise doubled on PS2 (branch pair `0x002B07E4`/`0x002B07EC`)
  — it survives from the shipped binary into m2c, not the other way round.
* **Case 6 ordering quirk.** All three dereference `trigger->data` at the
  view-format gate before testing it for NULL: GC `:433` vs `:436`; PS2
  `lbu` @`0x002B0A4C` region vs the check after `0x002B0A58`; PC
  `cmp byte [ecx],0` @`0x0044B008` vs `test eax,eax` @`0x0044B010`. The
  `<= 1` skip of the deref is `sltiu $1,$2,2` (PS2) ≡ `cmp eax,1; jbe` (PC)
  ≡ `> 1` short-circuit (GC).
* **Case 10 helper usage.** PC calls `SharedSlaveHolder_Get` (`0x0044AAB0`)
  at `0x0044B21A`; PS2 inlines Get **inside case 10** (`0x002B0E00`–`0x002B0EB4`:
  sets `active` with `sw $19,0($2)`, `jal ThingManagerList_InitAllSlavesNow`
  @`0x002B0E88` with delay-slot `addiu $4,$2,4` passing `(u8 *)shared + 4`,
  scratch store `sb $20,0x1D($3)`) — while the standalone PS2
  `SharedSlaveHolder_Get` (`0x002B11F0`) in turn inlines
  `InitAllSlavesNow`'s body. GC keeps all three as separate calls. This
  three-way duplication is the strongest single identifier for the trio
  (see `ThingManagerList_InitAllSlavesNow`).
* **Cold-path epilogue duplication (PC only).** Cases 10
  (`0x0044B222`–`0x0044B230`) and 99 (`0x0044B273`–`0x0044B27B`) each carry
  their own `pop esi / pop ebx / add esp,0x178 / ret` after MSVC moved them
  out of line — in case 10 the `pop esi` is even interleaved *before* the
  scratch-byte store (`pop esi` @`0x0044B225`, `mov byte [ecx+0x1d], al`
  @`0x0044B226`, with `ecx` loaded from `esi` while `esi` is still live);
  every other path converges on `0x0044AFF2`. GC/PS2 have a single shared
  tail.
* **`id` width.** `trigger->id` is `u16`, but every use site narrows to a
  byte on all platforms: PC `mov al/bl, [esi+0x12]`, GC/PS2 `CreateSlave`'s
  `u8 id` parameter and the byte compare against `0x82`.
