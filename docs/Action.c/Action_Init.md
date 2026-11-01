# `Action_Init`

## Signature

* **GC** `void Action_Init(void)` — `0x800058E8`, size `0xc0`
  (`HWVX-GC-Decomp/src/Action.c`).
* **PS2** **no symbol** — this name does not exist in
  `HWVX-PS2-Decomp/symbol_addrs.txt`; the work it does appears inside
  `Action_Create` (0x24F400) instead.
* **PC** not yet matched.

## Arguments

None.

## Execution

Zeroes the module's action registry:

```c
void Action_Init(void) {
    extern void *m_Actions_800DBC80[80];   /* 0x140 bytes */
    for (i = 0; i < 80; i++)
        m_Actions_800DBC80[i] = NULL;
}
```

The loop is 0xc0 bytes on GC because CodeWarrior emits one store per iteration
(no unrolling at `-O4 -fp` settings used here); `Action_ReleaseAllActions`
does the same clear with `memset`.

## Version differences

* **GC-only name.** PS2's retail ELF has no `Action_Init`; PS2 performs the
  identical registry clear (0x140 bytes at `D_518D50`, dword-unrolled) inside
  `Action_Create` — see `Action_Create.md` for the disassembly and why the
  pairing is `Action_Create` ↔ `Action_Init` on PS2 rather than
  `Action_Create` ↔ `fn_800957D4` as on GC.
* Practical consequence: PS2's action table starts zeroed at module init
  without any call from the GC-style `Action_Create` stub path; PS2 docs should
  treat "no `Action_Init` symbol" as **merge into `Action_Create`**, not as
  "the loop was deleted".
