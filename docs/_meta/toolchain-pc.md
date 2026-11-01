# PC toolchain — Microsoft Visual C++ 6.0 (via wibo), decompnet harness

The PC decomp (`HWVX-PC-Decomp`) targets `orig/hotwheels.exe`
(sha1 `91af8432d6903371f724c4c2924c4ba51cc9eb73`, image base `0x400000`).

* Harness: [decompdecomp](https://github.com/n-delic/decompdecomp) PC variant
  (decompnet / objdiff-cpp style). `tools/configure.py hotwheels --msvc <VC++ dir> --wibo <wibo>`
  → `build.ninja`; each function is compiled to its own object
  (`build/hotwheels/src/auto/text_<addr>.obj`) and diffed by objdiff
  (`objdiff.json`).
* Compiler: `CL.EXE` from `msvc/6.0` in the decompnet toolchain cache
  (`~/.local/share/decompnet/toolchains/msvc/6.0/`), executed through wibo on Linux.
* `config/hotwheels/functions.json` — analysis of the exe: 165 text units,
  **1643 functions**, each with address/size/relocations.
* `config/hotwheels/file_flags.json` — per-unit extra flags; so far four units
  take `/Oy-` (disable frame-pointer omission), indicating those functions were
  compiled with frame pointers in the retail build.
* `config/hotwheels/file_compilers.json` — per-unit compiler choice (empty: default MSVC).

Notes for later decomp work:

* Current `src/` only contains `src/auto/*` machine drafts; named files
  mirroring GC must be introduced as functions are identified.
* MSVC 6.0 emits SEH tables; the PC functions.json relocations reference data
  like `_D_004806E0` — these need `#define`d/typed globals for matching.
* Game is C; prefer `.c` sources where MSVC's C front end suffices, keeping
  `.cpp` only where name-mangling evidence proves C++ (unlike GC/PS2, the PC
  harness currently uses `.cpp` for auto units).
