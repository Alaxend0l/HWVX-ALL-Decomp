# `Action_Create`

## Signature

* **GC** `void Action_Create(void)` — `0x800058C4`, size `0x20`
  (`HWVX-GC-Decomp/src/Action.c`).
* **PS2** `void Action_Create(void)` — `0x24F400`, size `0x4c`
  (`HWVX-PS2-Decomp/symbol_addrs.txt`; m2c draft
  `HWVX-PS2-Decomp/progress/best/Action_Create.c`).
* **PC** not yet matched to this name (units are `_func_004XXXXX` placeholders).

## Arguments

None. Both platforms reach the module's action table through file-static
(or absolute) data, not through a parameter.

## Execution

**GC:** the function does nothing but tail-branch to a shared initializer —

```c
void Action_Create(void) {
    fn_800957D4();          /* not yet identified; 0x20 bytes of call glue */
}
```

`fn_800957D4` lives far outside this module (in the object/allocator area), so
"creating" an action on GC means constructing the *record* (a 0x18-byte `Action`
struct) somewhere else and handing it to `Action_AddAction`.

**PS2:** there is no delegation — the function *is* the initializer, and it
clears the table in dword-unrolled groups of 8 (`asm/nonmatchings/cod/Action_Create.s`
at `0x24F400`):

```c
void Action_Create(void) {
    for (i = 0; i < 10; i++)            /* byte offset += 0x20 per group */
        for (j = 0; j < 8; j++)
            *((s32 *)&D_518D50 + i * 8 + j) = 0;   /* 0x140 bytes total */
}
```

The `m2c` draft rendered this as "eight stores at `0x0..0x1C` per iteration",
which *looks* like record-granular zeroing — but `Action_Heartbeat` (see its
doc) indexes the same symbol with **stride 4** and 0x50 slots, so `D_518D50`
is GC's array of 80 action *pointers*, and `Action_Create` on PS2 is a plain
zero-the-registry.

## Version differences

* **GC delegates, PS2 is self-contained.** GC's `Action_Create` is a 0x20-byte
  stub forwarding to `fn_800957D4`; PS2 inlines the zeroing loop, which is why
  PS2's copy is `0x4c` bytes (a 2.4× size delta) and why PS2's ELF has **no
  `Action_Init` symbol** at all — its job got folded into `Action_Create`.
* **Record layout is the same on both platforms.** PS2's `Action_AddAction` and
  `Action_Heartbeat` use GC's field offsets verbatim — `type` at `0x0`, `data`
  at `0x4`, `count` at `0x10`, `delayInit` at `0xC`, `delay` at `0x14` — and
  index the table with stride 4 / 80 slots. The table itself is
  `D_518D50` on PS2 (absolute, high-half load) vs
  `m_Actions_800DBC80` on GC.
* **PS2 uses absolute data (`D_518D50`) rather than a named static** because the
  PS2 symbol table carries no game-data globals — every doc of this family
  should quote the `D_…` address when naming the table.
