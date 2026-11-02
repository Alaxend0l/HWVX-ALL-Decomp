# `ActionCodeVariableModifyValue`

## Signature

* **GC** `s32 ActionCodeVariableModifyValue(u32 *data)` — `0x80005DB4`, size `0x244`
  (`HWVX-GC-Decomp/src/Action.c:216`).
* **PS2** `s32 ActionCodeVariableModifyValue(s32 *data)` — `0x0024EDC0`, size `0x2A4`
  (`asm/nonmatchings/cod/ActionCodeVariableModifyValue.s`; no m2c draft).
* **PC** `s32 __cdecl ActionCodeVariableModifyValue(u32 *data)` — `0x004012F0`, size `0x230`
  (unit `text_00401090`; body `0x004012F0`–`0x0040149F`, epilogue `mov al,1; pop ebx; pop ecx; ret`
  at `0x004014A0`, then two MSVC switch tables `0x004014A8`–`0x004014FF`; next function starts
  `0x00401520`). **PC-CONFIRMED** (full disassembly verified).

## Arguments

`data` = record's `data` words:

| word | meaning |
|---|---|
| `data[0]` | opcode: `15` add, `16` sub, `45` mul, `46` div, `17` assign (anything else → no-op) |
| `data[1]` | code-variable id |
| `data[2]` | when `data[1] < 100`: object count; else: object pointer (slot 0) |
| `data[3..]` | when `data[1] < 100`: object pointers `[0..count)`, then at `data[count+3]` an operand selector `{0,1,2}` with operand words at `data[count+4]`/`data[count+5]`; else: `data[3]` selector, operand at `data[4]`/`data[5]` |

Returns `1` always (all paths, including the empty loop).

## Execution

```c
s32 ActionCodeVariableModifyValue(u32 *data) {
    f32 value, operand, cur;
    u32 i;

    if (data[1] < 100) {                      /* per-object variant */
        switch (data[data[2] + 3]) {          /* operand source */
        case 0: operand = *(f32 *)&data[data[2] + 4]; break;
        case 1: operand = *(f32 *)(*(u8 **)data[data[2] + 4] + 4); break;
        case 2: operand = GetCodeVariable(data[data[2] + 4], (u8 *)data[data[2] + 5]); break;
        }
        for (i = 0; i < data[2]; i++) {
            cur = GetCodeVariable(data[1], (u8 *)data[i + 3]);
            switch (data[0]) {
            case 15: cur += operand; break;
            case 16: cur -= operand; break;
            case 45: cur *= operand; break;
            case 46: if (operand) { cur /= operand; } break;   /* zero guard */
            case 17: cur = operand; break;
            }
            SetCodeVariable(data[1], (u8 *)data[i + 3], cur);
        }
    } else {                                  /* single-target variant */
        value = GetCodeVariable(data[1], (u8 *)data[2]);
        switch (data[3]) {                    /* operand source, fixed slots */
        case 0: operand = *(f32 *)&data[4]; break;
        case 1: operand = *(f32 *)(*(u8 **)data[4] + 4); break;
        case 2: operand = GetCodeVariable(data[4], (u8 *)data[5]); break;
        }
        switch (data[0]) {
        case 15: value += operand; break;
        case 16: value -= operand; break;
        case 45: value *= operand; break;
        case 46: if (operand) { value /= operand; } break;
        case 17: value = operand; break;
        }
        SetCodeVariable(data[1], (u8 *)data[2], value);
    }
    return 1;
}
```

Selector `1` is a double-deref cell: `*(f32 *)(*ptr + 4)` — load the pointer stored in
`data[..]`, then read the `f32` at offset `+4` of the pointee (another code-variable record).

## PC match

`0x004012F0` — `push ecx; push ebx; push esi; mov esi,[esp+0x10]` (`data`).

* Branch split `mov eax,[esi+4]` (`data[1]`), `cmp eax,0x64`, **`jae 0x004013F8`** (unsigned)
  — the `jae` proves the original comparison is **unsigned**, pinning `data[1]` as `u32` in
  all builds.
* Operand cascade (branch A) is source-order `0`, `1`, `2`:
  `sub ecx,0; je 0x401338` → `fld [esi+eax*4+0x10]` (literal `data[count+4]`);
  `dec ecx; je 0x401329` → `mov eax,[esi+eax*4+0x10]; mov ecx,[eax]; mov edx,[ecx+4];
  mov [esp+8],edx` (double-deref cell, moved through GPRs);
  fallthrough `dec ecx; jne 0x401340` → 2 → `push data[count+5]; push data[count+4];
  call 0x0044F5C0` (GetCodeVariable) @`0x0040131F`. Operand lands in stack slot `[esp+8]`.
* Loop `lea edi,[esi+0xc]` (`&data[3]`), count re-read every iteration
  (`mov eax,[esi+8]` @`0x00401340`), body `push data[i+3]; push data[1];
  call 0x0044F5C0` @`0x00401358`; the opcode switch is the classic two-level MSVC jump table:
  `add eax,-0xf; cmp eax,0x1f; ja default` + byte index map `mov dl,[eax+0x4014C0]` +
  `jmp [edx*4+0x4014A8]`. Arms: `fadd` @`0x0040137D`, `fsub` @`0x0040138B`, `fmul` @`0x00401399`,
  div guard `fld operand; fcomp dword ptr [0x00477000]` (0.0f global) `fnstsw ax;
  test ah,0x41; jnp` → `fdiv` (zero **or NaN/unordered → skip**, see below),
  and `fld operand; fstp cur` for opcode 17; `call 0x0044F6B0` (SetCodeVariable) @`0x004013DA`.
* Branch B @`0x004013F8`: `call 0x0044F5C0` for `GetCodeVariable(data[1],data[2])`, selector
  `mov eax,[esi+0xc]` with fixed offsets (`fld [esi+0x10]` / `mov eax,[esi+0x10]; mov ecx,[eax];
  fld [ecx+4]` / `call 0x0044F5C0` @`0x0040141F`), a **second** byte-map/jump-table opcode switch
  (`mov dl,[eax+0x4014E8]; jmp [edx*4+0x4014D0]`), `fadd`/`fsub`/`fmul` @`0x00401458`/`464`/`470`,
  the same `fcomp [0x00477000]` div guard @`0x00401483`, assign @`0x0040148E`,
  `call 0x0044F6B0` @`0x00401497`, then the single shared `return 1`.
* PC call targets: GetCodeVariable `0x0044F5C0`, SetCodeVariable `0x0044F6B0`.

## Version differences

* **Branch test**: PC `cmp eax,0x64; jae` @`0x004012FA` and PS2 `sltiu $1,$4,0x64` @`0x0024EDE8`
  are both **unsigned**, confirming `data[1]` is an unsigned id (a signed `<100` would have given
  PS2 `bge` / PC `jge`); GC `cmpwi`-style signed/unsigned cannot be read from the draft.
* **Two opcode switch copies** exist in every build (one per branch): PS2 emits two FP cascades
  (`add.s` @`0x0024EEBC`/`0x0024EFE0`, `sub.s` @`0x0024EEC4`/`0x0024EFE8`,
  `mul.s` @`0x0024EECC`/`0x0024EFF0`, `div.s` @`0x0024EEF0`/`0x0024F014`) with the FP ops sitting
  in **branch delay slots**; PC emits two byte-map/jump-table switches
  (`0x004014A8`+`0x004014C0` and `0x004014D0`+`0x004014E8`) — a 1:1 structural fingerprint that
  the GC source really has two duplicated `switch (data[0])` bodies.
* **Divide guard semantics differ on NaN**: PS2 tests `c.eq.s` against zero
  (@`0x0024EED8`, `0x0024EFFC`) — with a **NaN operand the compare is false, so the divide
  executes** (result NaN). PC uses MSVC's `fcomp [0x00477000]` + `fnstsw` + `test ah,0x41;
  jnp` idiom, whose parity check treats **unordered (NaN) as guard-fail, so the divide is
  skipped**. PC and PS2/GC therefore diverge for NaN operands — a deliberate-looking guard
  difference to preserve in decomp (write `if (operand != 0.0f)`; on PC the compiler's own
  NaN handling created the skew).
* **Named vs raw calls**: PS2 has named `jal GetCodeVariable` @`0x0024EE4C`, `0x0024EE6C`,
  `0x0024EF40`, `0x0024EF90` and `jal SetCodeVariable` @`0x0024EF18`, `0x0024F034` (two operand
  gets + one get per loop + two sets — matching the source exactly); PC/GC only show raw
  `call`/`bl` targets `0x0044F5C0`/`0x0044F6B0`.
* **Loop shape**: PC and PS2 re-read `data[2]` (count) on every iteration (PC
  `mov eax,[esi+8]` @`0x00401340` / `mov ecx,[ebx+8]` @`0x004013DF`) — matching the GC draft's
  `i < data[2]` test being re-evaluated; a hoisted count would let scripts mutate the count
  mid-loop with different results.
* PC keeps the `value` operand float in a **stack slot** `[esp+8]` written through GPRs
  (`mov [esp+8],edx` @`0x00401332`) — MSVC /G5 register-pressure artifact; GC/PS2 keep it in a
  register/FPU register.
