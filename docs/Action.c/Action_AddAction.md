# `Action_AddAction`

## Signature

* **GC** `s32 Action_AddAction(void *arg0)` — `0x80005A9C`, size `0x148`
  (`HWVX-GC-Decomp/src/Action.c:119`-`160`).
* **PS2** `s32 Action_AddAction(void *arg0)` — `0x24F250`, size `0xb0`
  (`asm/nonmatchings/cod/Action_AddAction.s`).
* **PC** not yet matched.

## Arguments

`arg0` — pointer to a fully built action *record* (the `Action` struct
documented in `_README.md` / `Action_Create.md`: `type`, `data`, `count`,
`delayInit`, `delay`). Ownership: the record is **borrowed**, not copied — the
caller keeps it (e.g. a static config blob) and the registry stores the pointer.

## Execution

Register the record in the first free slot and decide whether it self-
destructs:

```c
for (i = 0; i < 80; i++)
    if (m_Actions_800DBC80[i] == NULL) break;
if (i >= 80) {
    Dprintf("The max number of actions has been exceeded ... %d ...", 80);
    return 1;
}
if (action->count != 0xFFFFFFFF) action->count--;
m_Actions_800DBC80[i] = action;
m_Actions_800DBC80[i]->delay = m_Actions_800DBC80[i]->delayInit;
return action->count != 0;        /* 1 = still pending, 0 = fully consumed */
```

Return value tells the *level-script* caller whether to queue more copies:
`0` means this record's `count` reached 0 and the queued instance is final.

**PS2** (decoded from `0x24F250`): identical control flow, at identical field
offsets — scan 0x50 slots (`D_518D50`, stride 4), on overflow
`Dprintf("…", 0x50)` with the format string at `D_002CDAF0` and return 1;
`count` at `0x10` (decrement unless `-1` = infinite); store at `slot[i]`;
`delay(0x14) = delayInit(0xC)`; return `count != 0`.

## Version differences

* **Semantics match exactly**; the size delta 0x148 → 0xb0 is the debug path:
  GC's build keeps more of the `Dprintf` argument marshalling live (the
  readonly-string region around this function, `#pragma readonly_strings on`
  in the source, exists for that string), while PS2's MIPS emits a compact
  `jal Dprintf` with the format address in `$4`.
* Only cosmetic data differences: GC names the table `m_Actions_800DBC80` and
  the string is GC-side static data; PS2 has `D_518D50` and `D_002CDAF0`.
* `count == 0xFFFFFFFF` as the "unlimited" sentinel is the same constant on
  both (`-1` compare in the PS2 asm) — worth pinning in any type-ID table doc.
