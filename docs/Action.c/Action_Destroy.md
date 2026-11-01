# `Action_Destroy`

## Signature

* **GC** `void Action_Destroy(void)` — `0x800058E4`, size `0x4`
  (`HWVX-GC-Decomp/src/Action.c`).
* **PS2** `void Action_Destroy(void)` — `0x24F3F0`, size `0x8`
  (`HWVX-PS2-Decomp/symbol_addrs.txt`).

## Arguments

Declared with none on both platforms, and it is *called* by the action
teardown path with the record pointer in `$a0`/`r4` — the C signature is
faithful to the source (unused parameter), not to the ABI.

## Execution

* **GC:** literally empty — `jr ra; nop`, 4 bytes. Records are pool-allocated
  elsewhere, so freeing an action is a no-op here; the registry clears slots
  directly (`Action_Heartbeat`, `Action_ReleaseAllActions`).
* **PS2:** 8 bytes — one instruction plus the return slot (MIPS needs the
  8-byte minimum frame-less function). Same semantics: nothing.

## Version differences

* **Size 4 → 8 is an ISA artifact, not a code change.** PPC encodes a bare
  `blr` in one word; MIPS cannot (no delay-slot-free single-word return), so an
  empty C function is 4 bytes on GC and 8 on PS2. Treat this pair as **identical
  behaviour** — the one place in this family where the size delta in
  `docs/_meta/version-differences.md` should *not* be read as a logic difference.
