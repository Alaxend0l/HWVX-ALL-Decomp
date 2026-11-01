# `ActionMusicSetSong`

## Signature

* **GC** `s32 ActionMusicSetSong(void)` — `0x8000F8CC`, size `0x8`
  (`HWVX-GC-Decomp/src/AudioManager.c:720`).
* **PS2** `s32 ActionMusicSetSong(void)` — `0x00256250`, size `0x8`
  (m2c draft `progress/best/ActionMusicSetSong.c`).
* **PC** not yet matched (no PC symbols to match against yet).

## Execution

```c
s32 ActionMusicSetSong(void) {
    return 1;
}
```

**No-op stub** — same kind as the five-stub block documented in
`ActionSpeakerChangeMode.md`, but placed in the **audio link region**
(GC `0x8000F8CC`, next to the `AudioManager`/`ActionMusic*` code; PS2
`0x00256250`) rather than with the other five, so in the GC split it lives in
`AudioManager.c`. Scripted "change the background song" actions are inert in
the shipped builds; music changes that *work* go through
`ActionMusicChange` (GC 0x2EC → PS2 0x424, real body).

## Version differences

* 0x8 → 0x8, byte-identical stub shape (`bl`+`li r3,1` vs
  `jr $31`+`addiu $2,1`).
* Its neighbouring real handler `ActionMusicChange` is the size-outlier to
  keep in the same doc cluster (0x2EC/0x424) so nobody mistakes the stub for a
  link casualty.
