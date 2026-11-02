# `ActionSwapAIScript`

## Signature

* **GC** `s32 ActionSwapAIScript(u32 *action)` — `0x80006498`, size `0x104`
  (`HWVX-GC-Decomp/src/Action.c:525`).
* **PS2** `s32 ActionSwapAIScript(s32 *action)` — `0x0024E740`, size `0x118`
  (`asm/nonmatchings/cod/ActionSwapAIScript.s`; no m2c draft).
* **PC** `s32 __cdecl ActionSwapAIScript(u32 *action)` — `0x00401940`, size `0xC0`
  (unit `text_00401090`; body to `0x004019ED`, `mov edi,edi` filler at `0x004019EE`, jump table
  `0x004019F0`–`0x004019FF`). **PC-CONFIRMED** (full disassembly verified).

## Arguments

`action` = record's `data`: `action[0..3]` = the four new script IDs, `action[4]` = vehicle
list length, `action[5 + i]` = TileCar spawn pointer. Each vehicle's `+0x40..0x4C`
(`scripts[4]`) is overwritten wholesale; `+0x28` is the creature (AI car) pointer.

## Execution

```c
s32 ActionSwapAIScript(u32 *action) {
    for (i = 0; i < action[4]; i++) {
        vehicle = (u8 *)action[i + 5];
        vehicle->scripts[0] = action[0];     /* +0x40..0x4C — ALWAYS written, */
        vehicle->scripts[1] = action[1];     /* even when no creature exists   */
        vehicle->scripts[2] = action[2];
        vehicle->scripts[3] = action[3];
        creature = *(u8 **)(vehicle + 0x28);
        if (creature != NULL) {
            switch (GameSettings_GetDifficulty()) {   /* cases 0..3 only */
            case 0: ai->script_slot = action[0]; break;
            case 1: ai->script_slot = action[1]; break;
            case 2: ai->script_slot = action[2]; break;
            case 3: ai->script_slot = action[3]; break;
            }
            /* ai = *(void **)(creature + 0x94); script slot at ai + 0xD0 */
            AI_Car_Reset(creature + 0x94);            /* address of the +0x94 slot */
        }
    }
    return 1;
}
```

Note the odd dual addressing of `creature + 0x94`: the `+0xD0` store **dereferences** it
(`+0x94` holds the AIPlayer pointer) while `AI_Car_Reset` receives the **address** of the
slot — as written in the GC source; PC shows both uses (`mov eax,[edi+0x94]; mov [eax+0xD0],…`
vs `lea/jmp add edi,0x94; push edi`).

## PC match

`0x00401940` — `mov esi,[esp+0xc]`; count `[esi+0x10]` (`action[4]`);
`lea ebx,[esi+0x14]` = `&action[5]`.

* Scripts copied **before** the creature check: `[eax+0x40..0x4C] = action[0..3]`
  @`0x0040195F`–`0x00401973`, then `test edi,edi; je 0x004019D3` jumps to the loop tail —
  creature-null skips only switch + reset ✓ GC order (534–537 before 539).
* `call 0x0042AF90` (GetDifficulty) @`0x00401978`, no args; `cmp eax,3; ja 0x004019C4` —
  the **default arm skips the store but still resets** ✓ GC has only cases 0–3.
* `jmp dword ptr [eax*4+0x004019F0]` 4-entry jump table (`0x401989`/`0x401993`/`0x4019A4`/
  `0x4019B5`). Case 0 and case 3 are **tail-merged** into a common store
  `mov ecx,[esi+0xc or 0x18]; mov [eax+0xD0],ecx` @`0x004019BE` (MSVC merge of the two
  arms that read `action[0]`/`action[3]` through different offsets); cases 1/2 are
  self-contained @`0x00401993`/`0x004019A4`.
* Reset on **all** creature!=NULL paths: `add edi,0x94; push edi; call 0x00402EA0`
  (AI_Car_Reset).
* Loop tail re-reads the count (`mov eax,[esi+0x10]`) before `inc`/`cmp`;
  `mov al,1; pop …; ret`; hotpatch `mov edi,edi` at `0x004019EE`.

## Version differences

* **Difficulty fetch**: PS2 zero-extends a difficulty **byte** with
  `andi $3,$2,0xFF` @`0x0024E7A0` (byte-global load idiom) and cascades **descending**
  `3, 2, 1`, `beqz 0` with constants staged in registers; PC calls the function and trusts
  `eax` (`cmp eax,3; ja`); GC draft shows the plain `switch`. All agree on semantics.
* **`sw $2,0xD0(...)` stores sit in branch/call delay slots** on PS2 (staged cascade), and
  `AI_Car_Reset`'s argument is staged as `addiu $4,$18,0x94` **inside the call's delay
  slot** — the classic R4000 trap for decomp order sensitivity; PC has no delay slots, so
  the C order above is only faithful on MIPS if the staging survives.
* **Code size**: PC `0xC0` is roughly half of GC `0x104`/PS2 `0x118` thanks to the jump
  table + case 0/3 tail-merge; PS2 is the largest due to the staged descending cascade.
* Count re-read each iteration in **both** PC (`mov eax,[esi+0x10]`) and PS2
  (`lw $x,0x10($19)`) — matches GC `i < action[4]` re-evaluation.
