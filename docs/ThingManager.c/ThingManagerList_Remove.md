# `ThingManagerList_Remove`

Unlinks `thing` from whichever doubly-linked list it hangs on (active list,
free list, or a private child list). Does **not** clear the list head when
the last element is removed — the head just becomes `NULL` via the `prev`
path.

## Signature

* **GC** `void ThingManagerList_Remove(void **arg0, void *arg1)` —
  `0x80053FA0`, size `0x48` (`HWVX-GC-Decomp/src/ThingManager.c:99`).
* **PS2** — `0x002B18D0`, size `0x40`
  (`asm/nonmatchings/cod/ThingManagerList_Remove.s`).
* **PC** — `0x0044A7C0`, extent `0x50` (body ends `ret` @`0x0044A80E`), in
  unit `text_0044A0D0`.

## Arguments

`arg0` = address of the list-head slot the thing is on; `arg1` = the thing.
Touches only `prev` (`0xA0`) and `next` (`0xA4`).

## Execution

```c
void ThingManagerList_Remove(void **arg0, void *arg1) {
    void *temp_r5;
    void *temp_r5_2;

    temp_r5 = M2C_FIELD(arg1, void **, 0xA0);         /* prev   */
    if (temp_r5 == NULL) {
        *arg0 = M2C_FIELD(arg1, void **, 0xA4);       /* head = next */
    } else {
        M2C_FIELD(temp_r5, void **, 0xA4) = (void *)M2C_FIELD(arg1, void **, 0xA4);
    }
    temp_r5_2 = M2C_FIELD(arg1, void **, 0xA4);       /* next — re-read */
    if (temp_r5_2 != NULL) {
        M2C_FIELD(temp_r5_2, void **, 0xA0) = (void *)M2C_FIELD(arg1, void **, 0xA0);
        M2C_FIELD(arg1, void **, 0xA4) = NULL;
    }
    M2C_FIELD(arg1, void **, 0xA0) = NULL;            /* unconditional */
}
```

The `temp_r5` / `temp_r5_2` locals are m2c reconstructions of a real
register-reload pattern (see below) — the source almost certainly was written
with the same shape.

## PC match

```
0x0044a7c0: mov eax, [esp + 8]        ; arg1
0x0044a7c4: mov ecx, [eax + 0xa0]     ; prev
0x0044a7ca: xor edx, edx              ; NULL constant
0x0044a7cc: cmp ecx, edx
0x0044a7ce: push esi                  ; (flags survive the push)
0x0044a7cf: jne 0x44a7df              ; prev != NULL -> else
0x0044a7d1: mov ecx, [eax + 0xa4]     ; next
0x0044a7d7: mov esi, [esp + 8]        ; arg0 (shifted by the push)
0x0044a7db: mov [esi], ecx            ; *arg0 = next     (head path)
0x0044a7dd: jmp 0x44a7eb
0x0044a7df: mov esi, [eax + 0xa4]     ; next
0x0044a7e5: mov [ecx + 0xa4], esi     ; prev->next = next  (else path)
0x0044a7eb: mov ecx, [eax + 0xa4]     ; next — RE-READ (temp_r5_2)
0x0044a7f1: cmp ecx, edx
0x0044a7f3: je  0x44a807
0x0044a7f5: mov esi, [eax + 0xa0]     ; prev — RE-READ
0x0044a7fb: mov [ecx + 0xa0], esi     ; next->prev = prev
0x0044a801: mov [eax + 0xa4], edx     ; arg1->next = NULL
0x0044a807: mov [eax + 0xa0], edx     ; arg1->prev = NULL (both paths)
0x0044a80d: pop esi
0x0044a80e: ret
```

## Version differences

* **The reload quirk is on all three platforms**: after the first branch, all
  three *re-read* `arg1->next` (GC `temp_r5_2 = ...0xA4` line 109; PS2
  `lw $4,0xA4($5)` @`0x002B18F0`; PC `mov ecx,[eax+0xa4]` @`0x0044A7EB`) and
  PS2/PC then also re-read `arg1->prev` for the back-link (GC line 111
  `M2C_FIELD(arg1,0xA0)`; PS2 `lw $3,0xA0($5)` @`0x002B18FC`; PC
  `mov esi,[eax+0xa0]` @`0x0044A7F5`). A decomp that caches these values will
  not diff-match.
* **PS2 inverts and merges**: `bnez $6` @`0x002B18D4` takes the *else* path
  (prev != NULL) and the head path reaches the common tail through an
  unconditional `b` whose **delay slot is the head store itself**
  (`b .L002B18F0` / `sw $3,0x0($4)` @`0x002B18E0`/`0x002B18E4`). The final
  `arg1->prev = NULL` store lives in the return delay slot (`jr $31` /
  `sw $0,0xA0($5)` @`0x002B1908`/`0x002B190C`). Both branch delay slots that
  guard the second block are wasted `nop`s (`0x002B18D8`, `0x002B18F8`).
* **PC register pressure**: `arg1` stays in `eax` throughout, but `arg0` must
  be re-fetched through the pushed stack (`mov esi,[esp+8]` @`0x0044A7D7`,
  the original `[esp+4]` shifted by `push esi` @`0x0044A7CE`) — MSVC inserted
  the push *between* the `cmp` @`0x0044A7CC` and `jne` @`0x0044A7CF` because
  `push` does not disturb flags.
* Unconditional tail store `arg1->prev = NULL` (GC line 114) is outside the
  `if` on all three: PS2 reaches it even from the `beqz` at `0x002B18F4`
  (next == NULL jumps straight to the `jr` delay store).
