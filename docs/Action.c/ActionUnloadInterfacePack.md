# `ActionUnloadInterfacePack`

## Signature

* **GC** `void ActionUnloadInterfacePack(s32 *arg0)` — `0x80005D80`, size `0x34`
  (`HWVX-GC-Decomp/src/Action.c:210`).
* **PS2** `void ActionUnloadInterfacePack(s32 *arg0)` — `0x0024F070`, size `0x34`
  (`asm/nonmatchings/cod/ActionUnloadInterfacePack.s`; no m2c draft —
  `void`-returning, the drafts skipped it).
* **PC** not yet matched (match against GC — no PC symbols exist, see
  `docs/_meta/version-differences.md`).

## Arguments

Same record layout as `ActionLoadInterfacePack`: `*arg0` = pack **id**,
`arg0 + 1` (word ≡ byte 4) = pack **name**.

## Execution

```c
void ActionUnloadInterfacePack(s32 *arg0) {
    FileStreaming_ReleaseDataPack(
        FileStreaming_GetPackID((char *)(arg0 + 1), 1, *arg0), 1);
}
```

Name → internal pack index via `FileStreaming_GetPackID(name, type=1, id)`,
then release that index with the same type tag. No return value, no success
check — unload is best-effort.

## Version differences

* **Exactly the same size on both builds (0x34 = 0x34)** — one of the
  size-match outliers called out in `version-differences.md`'s "107 of 1,161"
  statistic. GC: 13 PPC words (two calls, arg setup); PS2: 13 MIPS words
  (`daddu $2,$4,$0` copy, `addiu $4,$2,4` / `lw $6,0($2)` / `addiu $5,1` →
  `jal GetPackID`; result → `jal ReleaseDataPack` with `addiu $5,1`).
* PS2 spills the frame (`addiu $29,-0x10`, `sd`/`ld` ra) because of the two
  nested calls; GC's PPC keeps the result in `r3` with a fixed frame — same
  nesting, different prologue.
* The type constant `1` (interface pack) appears twice on both sides; if a
  second pack type ever gets queued here, both builds change together.
