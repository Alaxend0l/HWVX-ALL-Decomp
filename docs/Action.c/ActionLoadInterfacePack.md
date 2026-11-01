# `ActionLoadInterfacePack`

## Signature

* **GC** `s8 ActionLoadInterfacePack(s32 *arg0)` — `0x80005D3C`, size `0x44`
  (`HWVX-GC-Decomp/src/Action.c:202`).
* **PS2** `s32 ActionLoadInterfacePack(s32 *arg0)` — `0x0024F0B0`, size `0x34`
  (m2c draft `progress/best/ActionLoadInterfacePack.c`).
* **PC** not yet matched.

## Arguments

`arg0` — record's `data`: `*arg0` = pack **id**, `arg0+1` (word, = byte
offset 4) = pack **name** string.

## Execution

```c
s8 ActionLoadInterfacePack(s32 *arg0) {
    return FileStreaming_LoadDataPack(arg0 + 1, 1, *arg0, 1) != -1;
}
```

Load the named data pack (type 1 = interface/HUD pack, sub-type 1) through the
file-streaming layer, and report success. `FileStreaming_LoadDataPack`
returns `-1` on failure, so the return is "did the load succeed".

The GC draft writes the success test as the classic no-branch idiom
`(s8)((u32)((-1 - temp_r3) | (temp_r3 + 1)) >> 0x1F)`; PS2's m2c draft as
`~temp != 0`. **Same predicate**: `temp != -1`.

## Version differences

* Size 0x44 → 0x34: GC keeps the branchless success fold live (the
  `sub`/`nor`-style ops + `extsb`), MIPS gets away with `nor`+`slot`.
* The name-pointer argument differs only in **C-level spelling**: GC treats
  `arg0` as `s32 *` so `arg0 + 1` is already byte+4; m2c's PS2 draft shows
  `arg0 + 4` because it types the base as `s8 *`. Emit one spelling per build,
  not a real semantic difference.
* Return typing: GC's symbol/prototype says `s8`, m2c defaults PS2 to `s32`
  (MIPS `jr` with `$2` — the caller (`Action_Heartbeat`) ignores the
  value anyway).
