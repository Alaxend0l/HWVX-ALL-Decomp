# `ActionPlaySoundSequence`

## Signature

* **GC** `s32 ActionPlaySoundSequence(void *arg0)` — `0x80006A04`, size `0xdc`
  (`HWVX-GC-Decomp/src/Action.c:773`).
* **PS2** `void ActionPlaySoundSequence(void *arg0)` — `0x0024E1A0`, size `0xd4`
  (`asm/nonmatchings/cod/ActionPlaySoundSequence.s`; m2c draft
  `progress/best/ActionPlaySoundSequence.c`).
* **PC** not yet matched (match against GC — no PC symbols exist).

## Arguments

`arg0` = record's `data`: `+0` = "play directly" gate, `+4` = queued flag,
`+8` = sound-sequence **id**.

## Execution

```c
s32 ActionPlaySoundSequence(void *arg0) {
    void SoundSequence_ClearQueue();
    void SoundSequence_Play(u32, s32);
    void SoundSequence_PlayQueued(u32, s32);
    extern u8 g_soundSeqVTable[];
    u32 id;
    void *thing;

    id = M2C_FIELD(arg0, u32 *, 8);
    if (M2C_FIELD(arg0, u32 *, 0) != 0) {         /* direct: start one sequence */
        if (id != 0) {
            if (M2C_FIELD(arg0, u32 *, 4) != 0)
                SoundSequence_PlayQueued(id, 0);
            else
                SoundSequence_Play(id, 0);
        }
    } else if (M2C_FIELD(arg0, u32 *, 4) != 0) {  /* queued-clear of the global queue */
        SoundSequence_ClearQueue();
    } else {                                      /* per-instance stop: walk things */
        for (thing = g_thingManager_DARYL.activeList; thing != NULL;
             thing = M2C_FIELD(thing, void **, 0xA4)) {
            if (M2C_FIELD(thing, u8 **, 0x8C) == g_soundSeqVTable &&
                M2C_FIELD(thing, u32 *, 0xB0) == id) {
                ((void (*)(void *))(M2C_FIELD(thing, void ***, 0x8C))[1])(thing);
            }
        }
    }
}
```

Three behaviours by (gate, queued): **direct** = play/queue-play id `0`;
**(0, ≠0)** = clear the global queue; **(0, 0)** = call vtable slot 1 on
every live sound-sequence *instance* whose `+0xB0` id matches — stop the
matching running sequences. Quirk to keep: in the **walk** branch there is
**no `id != 0` guard** (that guard is only in the direct branch), so a walk
with id `0` fires the slot-1 call on instances stored with id 0.

## Version differences

* **0xdc GC vs 0xd4 PS2**, and both sides **never return a value** despite
  the `s32` declaration: the PS2 tail (`0x24E25C` nop → `ld ra`/`lq`/`jr`)
  has **no `addiu $2,1`**, so `$2` is residue (`lw $2,4($4)` queue flag, the
  vtable fn ptr from `lw $2,4($4)` at `0x24E240`, …) on every path, and
  m2c drafts it `void`. **Do not "fix" this with `return 1`** — it breaks
  the size match on both builds.
* `id` is loaded in the **delay slot of the gate test** (`lw $16,0x8($4)`
  at `0x24E1B8`) — loaded once per call even on the walk path; GC hoists
  `id = ...` above the `if` for the same reason.
* The vtable-match constant resolves to the **data symbol** `D_002C92E0`
  (`lui`/`addiu` at `0x24E224/0x24E228`) on PS2; GC names the same table
  `g_soundSeqVTable[]`. When naming PS2 data, that symbol is the
  sound-sequence vtable; the walk base is `*(void **)0x6E4BA0` =
  `g_thingManager_DARYL.activeList`.
* **Register-reuse trap**: the vtable-check load doubles as the call-target
  base — `lw $4,0x8C($17)` feeds both the `bne $4,$3` gate *and*, after the
  id compare, `lw $2,4($4)` (`0x24E240`) + `jalr $2`, with the thing arg
  moved in the `jalr` delay slot (`daddu $4,$17,$0`). A decomp that spills
  the vtable to a temp is fine semantically but won't reproduce this shape.
* **Scheduler quirk on the ClearQueue arm**: `lui $1,(0x6E4BA0 >> 16)`
  (`0x24E208`) — the *upper half* of the walk's base-address load — is
  staged into the delay slot of `jal SoundSequence_ClearQueue`, i.e.
  scheduled onto a path that never uses it. Keep it if diffing exactly.
* The two per-thing gates jump to one shared continue label
  (`.L0024E24C`, nop-padded at `0x24E24C`), then `bnez $17,.L0024E220`
  do/while at the bottom (`0x24E254`) — the list head is separately guarded
  by `beqz $17,.L0024E25C` at `0x24E218` before entering.
* Direct-play calls pass `(id, 0)` as `daddu $4,$16,$0` + `daddu $5,$0,$0`
  **in the `jal` delay slot** — the zero second arg is a `daddu`, not an
  `addiu`, because it was scheduled behind the call.
* PS2 frame `0x30` (`sd ra` + `sq $16/$17`); GC's PPC uses its own fixed
  frame — both spill because of the calls, no semantic difference.
