# `ThingManagerList_InitAllSlavesNow`

Two-phase "slam reset" for one player's slave set. Phase 1 recycles every
thing in `list` tagged with `player`: calls the thing's `Release`
(vtable[1]), unlinks it from `list`, and pushes it onto the freeList for its
type. Phase 2 then immediately fires **every** trigger in a `SlaveList`
through `ThingManagerList_TriggerSlaves`, so the freed slaves are recreated
at the trigger positions in the same frame. On PC the sweep is not inlined
at all — the function is a thin wrapper that *calls*
`ThingManagerList_ReleaseSlavesBySlaveType(list, player)`; GC and PS2 inline
the equivalent loop instead.

This is the function `SharedSlaveHolder_Get` calls after claiming a holder
slot, and `ThingManagerList_TriggerSlaves` case 10 calls it with the
`SharedSlaves` record — see the Arguments section for why the second
parameter is `SharedSlaves + 4`.

## Signature

| Platform | Address | Size | Source |
| --- | --- | --- | --- |
| GC | `0x80055014` | `0x170` | `HWVX-GC-Decomp/src/ThingManager.c:504` |
| PS2 | `0x002B0380` | `0x140` | `HWVX-PS2-Decomp/asm/nonmatchings/cod/ThingManagerList_InitAllSlavesNow.s` (90 lines) |
| PC | `0x0044B310` | `0x80` | unit `text_0044AD00`, `HWVX-PC-Decomp/config/hotwheels/functions.json` |

GC prototype (m2c-derived, see file for the local type declarations at
`:505`–`:520`):

```c
void ThingManagerList_InitAllSlavesNow(void *list, void *slaves, u8 player);
```

## Arguments

* `list` — the same head-pointer slot used everywhere else in this module
  (a `ThingHeader **` whose pointee chain is walked via `next` at `0xA4`).
  It is both the list the sweep unlinks from and the list `TriggerSlaves`
  prepends new slaves onto. In practice this is
  `g_thingManager_DARYL.activeList` or a per-holder slave list.
* `slaves` — a `SlaveList`:

  ```c
  typedef struct {              /* GC ThingManager.c:517-520 */
      u32          count;       /* +0x00 */
      SlaveTrigger *triggers;   /* +0x04 */
  } SlaveList;
  ```

  This is exactly the `SharedSlaves` record viewed from offset 4: the
  record is `{u32 active@0; u32 count@4; SlaveTrigger *triggers@8}`, so
  `slaves = (u8 *)shared + 4`. The PS2 caller proves it: inside
  `ThingManagerList_TriggerSlaves` case 10, `jal
  ThingManagerList_InitAllSlavesNow` (`0x002B0E88`) is followed in the
  delay slot by `addiu $4, $2, 4` — `$2` is the `SharedSlaves` entry and
  the +4 adjusts it into the `SlaveList` view. Likewise
  `SharedSlaveHolder_Get` (PS2 `SharedSlaveHolder_Get.s`) passes
  `entry + 4`.
* `player` — `u8`, compared against `thing->slavePlayer` (byte at `0x83`,
  `0xEF` sentinel semantics documented in
  [`ThingManagerList_EmancipateSlaves.md`](ThingManagerList_EmancipateSlaves.md)).

## Execution

Reconstructed from GC `:504`–`:540` (the m2c output there is already close
to this form; `:521`–`:533` and `:534`–`:539` are verbatim):

```c
void ThingManagerList_InitAllSlavesNow(void *list, void *slaves, u8 player)
{
    SlaveList *sl = (SlaveList *)slaves;
    ThingHeader *thing, *next;
    ThingHeader **freeList;
    int i;

    /* phase 1: recycle this player's things */
    thing = *(ThingHeader **)list;                       /* :521 */
    while (thing != NULL) {                              /* :522 */
        next = thing->next;                              /* :523  (0xA4) */
        if (thing->slavePlayer == player) {              /* :524  (0x83) */
            thing->vtable[1](thing);                     /* :525  (0x8C[1]) */
            ThingManagerList_Remove(list, thing);        /* :526 */
            thing->flags &= ~0x200000;                   /* :527 */
            thing->flags |= 0x1000;                      /* :528 */
            freeList = g_thingManager_DARYL.freeList
                     + thing->type;                      /* :529  (0xA8) */
            ThingManagerList_Append(freeList, thing);    /* :530 */
        }
        thing = next;                                    /* :532 */
    }

    /* phase 2: fire every trigger immediately */
    for (i = 0; i < sl->count; i++) {                    /* :534 */
        Vector4 dir = {0.0f, 0.0f, 0.0f, 1.0f};          /* :535 */
        SlaveTrigger *trigger = &sl->triggers[i];        /* :536 */

        ThingManagerList_TriggerSlaves(list, (u8 *)trigger,
            (Vector4 *)trigger, (Vector4 *)&dir, &g_zeroVector, player);
    }                                                      /* :538 */
}
```

The phase-1 body is character-for-character the same sweep as
[`ThingManagerList_ReleaseSlavesBySlaveType`](ThingManagerList_ReleaseSlavesBySlaveType.md)
(GC helper at `:139`, loop `:145`–`:162`) — same byte-`0x83` compare, same
flag pair, same `freeList[thing->type]` append. The only source-level
difference inside GC is that the helper open-codes the 4-store append splice
at `:154`–`:160` while this function *calls* `ThingManagerList_Append` at
`:530`.

`SlaveTrigger` is 0x20 bytes (`pos` Vector4 `@0x00`, `type` u8 `@0x10`,
`id` u16 `@0x12`, pad `@0x14`, `data` `@0x18`) — the stride `add ebx, 0x20`
/ `addiu $20, 0x20` in both binaries matches. `TriggerSlaves` receives the
trigger twice, as its raw bytes (`arg2`) and as the `Vector4 *pos`
(`arg3`), with `b = &dir` = `{0,0,0,1}`, `c = &g_zeroVector` — the exact
arg pattern of [`ThingManagerList_TriggerSlaves.md`](ThingManagerList_TriggerSlaves.md)
with the trigger standing in for both position arguments.

## PC match

`0x0044B310`, 128 bytes, unit `text_0044AD00`. The fingerprint is a
3-relocation signature that is unique in the whole binary — this is the
**only** function that calls both `0x44A8A0`
(`ThingManagerList_ReleaseSlavesBySlaveType`) and `0x44AD00`
(`ThingManagerList_TriggerSlaves`) and touches `g_zeroVector`
(`0x00692350`) by pointer push:

| func offset | abs | reloc |
| --- | --- | --- |
| 17 | `0x0044B321` | `20, '_func_0044A8A0'` (call displacement) |
| 52 | `0x0044B344` | `6, '_D_00692350'` (imm32 in `push`) |
| 97 | `0x0044B371` | `20, '_func_0044AD00'` (call displacement) |

Full annotated disassembly (MSVC cdecl; entry `esp = E`, args at
`[E+4]`=list, `[E+8]`=slaves, `[E+0xC]`=player):

```asm
0044B310: mov   eax, [esp+4]            ; list (arg read before frame opens)
0044B314: sub   esp, 0x10               ; locals: dir at E-0x10..E-0x1
0044B317: push  ebp
0044B318: mov   ebp, [esp+0x20]         ; player   (E-0x14+0x20 = E+0xC)
0044B31C: push  esi
0044B31D: push  edi
0044B31E: push  ebp                     ; player
0044B31F: push  eax                     ; list
0044B320: call  0x44a8a0                ; ThingManagerList_ReleaseSlavesBySlaveType(list, player)
0044B325: mov   edi, [esp+0x2c]         ; slaves (E-0x24+0x2C = E+8) — stack re-read post-call
0044B329: mov   eax, [edi]              ; sl->count
0044B32B: add   esp, 8                  ; cdecl cleanup for helper call
0044B32E: xor   esi, esi                ; i = 0
0044B330: test  eax, eax
0044B332: jbe   0x44b383                ; count == 0 -> epilogue (ebx not pushed)
0044B334: push  ebx                     ; loop head has esp = E-0x20 from here
0044B335: xor   ebx, ebx                ; trigger byte offset (i * 0x20)
0044B337: mov   ecx, [edi+4]            ; sl->triggers          ; loop head
0044B33A: mov   edx, [esp+0x24]         ; list (E-0x20+0x24 = E+4) — arg re-read from stack
0044B33E: push  ebp                     ; arg6 = player
0044B33F: mov   eax, ebx
0044B341: add   eax, ecx                ; eax = &sl->triggers[i]   (0x20 stride via ebx)
0044B343: push  0x692350                ; arg5 = &g_zeroVector  (reloc +52)
0044B348: lea   ecx, [esp+0x18]         ; &dir  (E-0x28+0x18 = E-0x10)
0044B34C: push  ecx                     ; arg4 = &dir
0044B34D: push  eax                     ; arg3 = (Vector4 *)trigger
0044B34E: push  eax                     ; arg2 = (u8 *)trigger  (same pointer twice)
0044B34F: push  edx                     ; arg1 = list
0044B350: mov   dword ptr [esp+0x28], 0 ; dir.x = 0.0f   (E-0x38+0x28 = E-0x10)
0044B358: mov   dword ptr [esp+0x2c], 0 ; dir.y = 0.0f
0044B360: mov   dword ptr [esp+0x30], 0 ; dir.z = 0.0f
0044B368: mov   dword ptr [esp+0x34], 0x3f800000 ; dir.w = 1.0f
0044B370: call  0x44ad00                ; ThingManagerList_TriggerSlaves   (reloc +97)
0044B375: mov   eax, [edi]              ; count re-loaded EVERY iteration
0044B377: add   esp, 0x18               ; 6-arg cleanup
0044B37A: inc   esi
0044B37B: add   ebx, 0x20               ; sizeof(SlaveTrigger)
0044B37E: cmp   esi, eax
0044B380: jb    0x44b337
0044B382: pop   ebx
0044B383: pop   edi                     ; empty-count path skips 0x44B382
0044B384: pop   esi
0044B385: pop   ebp
0044B386: add   esp, 0x10
0044B389: ret
```

`0x0044B38A`–`0x0044B38F` are alignment padding up to the next function at
`0x0044B390`. Note the classic MSVC quirk: the four `dir` words are written
**after** `&dir` has already been pushed (`0x0044B34C` computes the address
via `lea`, the stores at `0x0044B350`–`0x0044B36B` land at the same
`E-0x10..E-0x1` range while `esp = E-0x38`) — legal, because
`TriggerSlaves` only reads `dir` after the call transfers control.

The PC `SharedSlaveHolder_Get` (`0x0044AAB0`) calls this function at
`0x0044AB02`, and another caller lives in the `0x00449CD0` helper at
`0x00449CE6` — matching the GC call map.

## PS2 cross-confirmation

The PS2 leaf (`.s`, 90 lines) matches structure for structure with GC,
register roles `$18`=list, `$17`=slaves, `$16`=player, `$20`=thing,
`$19`=next/i:

* Prologue frame is `0x70` with EE-paired saves: `addiu $29,$29,-0x70`
  then `sd`/`sq` of ra and `$16`–`$20` (`0x002B0384`–`0x002B039C`),
  mirrored by `lq`/`ld` at `0x002B04A0`–`0x002B04B4` and `jr $31` with
  `addiu $29,$29,0x70` in the delay slot (`0x002B04B8`/`0x002B04BC`).
* `lw $20,0($4)` + `beqz $20,...` at `0x002B03A4`/`0x002B03A8` — the
  `thing == NULL` pretest — with the player save `daddu $16,$6` hidden in
  the branch delay slot.
* Sweep loop `0x002B03B0`: `lbu $4,0x83($20)` + `andi $3,$16,0xFF` +
  `bne` with `lw $19,0xA4($20)` (next prefetch) in the delay slot
  (`0x002B03B4`–`0x002B03BC`); vtable[1] call via `lw $2,0x8C` /
  `lw $2,4` / `jalr $2` at `0x002B03C8`; `jal ThingManagerList_Remove` at
  `0x002B03D4` — **PS2 calls the Remove helper here**, only the splice
  tail is inlined.
* Flag RMW is split into two load/store pairs (`and` with `lui/ori
  0xFFDFFFFF` and `sw` at `0x002B03DC`–`0x002B03FC`, then `lw` /
  `ori 0x1000` / `sw` at `0x002B0400`–`0x002B0408`) with the freeList-base
  constants `lui/ori 0x7AB74` + `D_66A030` interleaved at
  `0x002B03E8`–`0x002B03F4`; slot address = `D_66A030 + 0x7AB74 +
  (lbu 0xA8 << 2)` (`0x002B040C`–`0x002B0418`) — byte-typed `type`.
* Append is open-coded (4-store splice, head store `sw $20,0($4)` at
  `0x002B0438`); backedge `bnez $19` at `0x002B0440` with `daddu $20,$19`
  in the delay slot.
* Trigger loop: `dir` is copied as one 16-byte quad — `lq` from
  `D_002C93B0` (= `{0.0, 0.0, 0.0, 1.0}`) at `0x002B0464`, `sq` to
  `sp+0x60` at `0x002B0474` — then the six args land in `$4`=`$18` (list),
  `$5`/`$6` = trigger (`addu $5,$2,$20` at `0x002B047C`, `daddu $6,$5` in
  the `jal` delay slot `0x002B0484` — the trigger passed twice as GC casts
  it), `$7` = `addiu $29,0x60` (&dir), `$8` = gp-abs `D_2DEED0`
  (= `g_zeroVector`), `$9` = `$16`; `jal ThingManagerList_TriggerSlaves`
  at `0x002B0480`. Loop control: `addiu $20,$20,0x20` (`0x002B0488`),
  bottom-guard `lw` of `count` + `sltu` + `bnez` at
  `0x002B0490`–`0x002B0498` — count re-read every iteration, like PC.

## Version differences

* **Phase-1 factoring.** PC emits a direct `call 0x44a8a0`
  (`ThingManagerList_ReleaseSlavesBySlaveType`) at `0x0044B320` — its only
  two identity-bearing relocations besides the `TriggerSlaves` call. GC
  (`:521`–`:533`) and PS2 (`0x002B03A4`–`0x002B0444`) inline the sweep
  loop instead, duplicating the helper body; net effect identical on all
  three.
* **Append call vs inline (inside GC itself).** This GC function *calls*
  `ThingManagerList_Append` at `:530`, while the GC
  `ReleaseSlavesBySlaveType` helper open-codes the splice at
  `:154`–`:160`. PS2 open-codes it here too (`0x2B041C`–`0x2B0438`); PC's
  copy of the splice lives inside the helper. PS2 still *calls*
  `ThingManagerList_Remove` (`jal` @`0x002B03D4`), consistent with its
  selective inlining policy.
* **Flag RMW granularity (PS2).** `flags &= ~0x200000` then `flags |=
  0x1000` is two separate load/modify/store pairs on PS2
  (`0x002B03DC`–`0x002B0408`, constants `0xFFDFFFFF` and `0x1000`), with
  the freeList-base `lui/ori/addiu` materialization interleaved between
  them. GC/PC keep each mask as one expression.
* **`type` field width.** PS2 indexes `freeList[type]` with `lbu` +
  `sll 2` (`0x002B040C`–`0x002B0414`), i.e. the PS2 `ThingHeader.type` is
  byte-sized; GC (`u32` load at `0xA8`, `:529`) and PC use a dword.
* **`dir` construction.** GC re-initializes the 4 scalars per iteration
  (`:535`); PS2 bulk-copies a pre-made constant quad `D_002C93B0` with
  `lq`/`sq` inside the loop (`0x002B0464`/`0x002B0474`); PC stores four
  immediates `0,0,0,0x3F800000` at `0x0044B350`–`0x0044B36B` **after**
  already pushing the struct's address.
* **Trigger argument marshalling.** Same six values everywhere; GC passes
  the trigger twice via two casts (`:538`), which MSVC honors as the same
  register pushed twice (`push eax` @`0x0044B34D` and `0x0044B34E`) and
  the EE honors as `$5`/`$6` (`addu` @`0x002B047C` + delay-slot `daddu`
  @`0x002B0484`). `g_zeroVector` is pushed by absolute address `0x00692350`
  on PC (the sole data reloc, `push` imm @`0x0044B343`), addressed
  gp-abs `D_2DEED0` on PS2, `&`-of-extern on GC.
* **Loop-bound re-read.** All three re-fetch `sl->count` from memory every
  iteration — PC `mov eax,[edi]` @`0x0044B375`, PS2 bottom-guard `lw`
  @`0x002B0490`, GC condition `i < sl->count` at `:534` — so a trigger
  that shrinks the list mid-loop is honored platform-wide.
* **Empty-count path (PC).** `jbe 0x44b383` @`0x0044B332` bypasses the
  `push ebx` at `0x0044B334`, so the `pop ebx` at `0x0044B382` is only
  reached by the loop path — MSVC keeps both epilogues balanced.
* **Duplication quirk (PS2 only).** PS2 `SharedSlaveHolder_Get`
  (`0x002B11F0`) does **not** call this function; it inlines the same
  two-phase body — `jal ThingManagerList_ReleaseSlavesBySlaveType`
  (`$4 = entry+4`, `$5 = 0xFF`) @`0x002B1278` plus `jal
  ThingManagerList_TriggerSlaves` @`0x002B12B4` — so PS2 carries a second
  copy of the trigger loop. PC's Get *does* call it
  (`call 0x44b310` @`0x0044AB02`). Conversely PS2's `TriggerSlaves`
  case 10 `jal`s the standalone function @`0x002B0E88`. The standalone
  symbol exists on PS2 (`0x002B0380`, 0x140, with `.s`) and matches GC
  1:1; the Get duplication is a compiler/policy artifact to keep in mind
  when counting callers.

## Cross-references

* [`ThingManagerList_TriggerSlaves.md`](ThingManagerList_TriggerSlaves.md)
  — phase-2 callee; case 10 is the PC caller that proves the
  `SharedSlaves + 4` = `SlaveList` view.
* [`ThingManagerList_ReleaseSlavesBySlaveType.md`](ThingManagerList_ReleaseSlavesBySlaveType.md)
  — the phase-1 body as a standalone function (PC callee @`0x0044A8A0`).
* [`ThingManagerList_Remove.md`](ThingManagerList_Remove.md) /
  [`ThingManagerList_Append.md`](ThingManagerList_Append.md) — the
  unlink/push pair the sweep performs (PC `0x0044A7C0` / `0x0044A790`).
* [`ThingManagerList_EmancipateSlaves.md`](ThingManagerList_EmancipateSlaves.md)
  — the opposite lifecycle transition (`0x83 = 0xEF` sentinel).
