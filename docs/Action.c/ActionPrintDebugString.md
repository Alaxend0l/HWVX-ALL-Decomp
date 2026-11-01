# `ActionPrintDebugString`

## Signature

* **GC** `s32 ActionPrintDebugString(u32 *action)` — `0x80005BE4`, size `0x158`
  (`HWVX-GC-Decomp/src/Action.c:145`).
* **PS2** `s32 ActionPrintDebugString(s32 *action)` — `0x0024F0F0`, size `0x154`
  (`asm/nonmatchings/cod/ActionPrintDebugString.s`; no m2c draft).
* **PC** not yet matched.

Despite the name, this is **not** a console-print: it is the "level-script
debug/object tweak" handler. `action` = record's `data` array:
`action[0]` = mode (0..8), `action[2]` = payload value, `action[1]` = a
countdown slot.

## Execution

```c
s32 ActionPrintDebugString(u32 *action) {
    if (action[0] == 0) {                      /* self-timing branch: countdown */
        if (action[1] == 0)
            action[1] = 0x3C;                  /* 60 ticks */
        action[1] -= 1;
    }
    else {
        action[1] = 0;
        for (i = 0; i < world[8]; i++) {       /* walk the object list */
            obj = world[0xC] + i * 0x60;       /* 0x60-byte object stride */
            if (obj[0x35] != 1)
                continue;                      /* only "active" objects */
            switch (action[0]) {
            case 1: ((u32 **)obj[0x28])[0x498 / 4] = action[2]; break;
            case 2: case 3: ((u32 **)obj[0x28])[0x494 / 4] = action[2]; break;
            case 4: InterfaceManager_SetPlayerMessage(action[2], 0); break;
            case 5: ((u32 **)obj[0x28])[0x49C / 4] = action[2]; break;
            case 6: ((u32 **)obj[0x28])[0x4A0 / 4] = action[2]; break;
            case 7: InterfaceManager_SetPlayerMessage(action[2], 1); break;
            case 8: InterfaceManager_SetPlayerMessage(action[2], 2); break;
            }
        }
    }
    return 1;
}
```

Modes 1/2/3/5/6 patch counters inside each object's "interface" sub-struct
(pointer at `obj+0x28`, fields `0x494`–`0x4A0`); 4/7/8 push a player message
for player 0/1/2. `0x3C` is the countdown default (60 = one second at 60 fps).

## Version differences

* **Sizes within 4 bytes (0x158 vs 0x154) and offsets identical** — GC and PS2
  share the object-list layout (stride `0x60`, gate byte `0x35`, inner pointer
  `0x28`, fields `0x494/0x498/0x49C/0x4A0`) and the loop bound at `world+0x8`
  (`lw $3,(0x707000)`; `lw $2,8($3)`; `sltu` in PS2 asm ≡ GC's
  `M2C_FIELD(g_worldData.world, u32 *, 8)`). The only PS2-side data naming is
  that absolute world pointer `0x707000`.
* PS2 compiles the mode switch to a **9-entry jump table**
  (`jtbl_002CDAC0`, guarded by `sltiu $1,$2,9`); GC's PPC uses
  `cmpi`/branch cascade (or its own table) — layout, not behaviour.
* **Codegen trap worth pinning**: in the PS2 case-7 arm there is no
  `addiu $5,…` — `$5` still holds **1** from the `obj[0x35] == 1` gate compare
  (`addiu $5,$0,1` at `0x24F144`) and is re-used as the `SetPlayerMessage`
  second argument. A hand-written PS2 decomp that hoists the gate constant out
  of the loop silently changes case 7's player index from 1 to whatever
  `addiu` survives. GC's PPC keeps it in a register pair and doesn't show this.
* The gate is `== 1` (not just non-zero) on both — keep `bne $2,$5`
  semantics, `!= 0` is wrong.
