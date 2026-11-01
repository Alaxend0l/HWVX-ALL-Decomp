# HWVX (Hot Wheels Velocity X) — decomp documentation

This directory documents the three parallel decomps of *Hot Wheels Velocity X*:

| Project | Platform | Status |
| --- | --- | --- |
| `HWVX-GC-Decomp` | GameCube | reference — nearly complete (309 split `src/*.c` files) |
| `HWVX-PS2-Decomp` | PlayStation 2 | splat9/objdiff harness; all code in one `cod` module, naming/splitting in progress |
| `HWVX-PC-Decomp` | Windows PC | decompnet harness (MSVC 6.0 objects); only `src/auto` so far |

GC is the **naming reference**: PS2 and PC functions should be named after their
GC counterparts (`Action_Destroy`, `Thing_Draw`, …), and source files should be
split to mirror GC's file list — no mega-files.

## Documentation layout

Conventions used by this tree:

* `docs/_meta/*.md` — project-level documentation (toolchains, version diffs, progress notes).
* `docs/<filename>/<function>.md` — one markdown file per decompiled function, grouped
  by the source file it lives in (named as in GC, e.g. `docs/Action.c/Action_Destroy.md`).
* `docs/<filename>/_README.md` — module overview: purpose, main data structures,
  function index.

Every function doc records, in this order:

1. **Signature** — per platform, if it differs (GC / PS2 / PC).
2. **Arguments** — name, type, meaning, which struct fields are read/written.
3. **Execution** — what the function does, step by step (control flow, side effects,
   callees, callers).
4. **Version differences** — how the PS2 and PC builds differ from GC (behavior,
   signature, presence, call shape, data layout).

## Index

* `_meta/toolchain-gc.md` — GameCube toolchain (CodeWarrior GC 1.3.2, dtk-template)
* `_meta/toolchain-ps2.md` — PS2 toolchain (MW MIPS C Compiler 2.4.1.01 / CodeWarrior PS2 R3.01, Wine)
* `_meta/toolchain-pc.md` — PC toolchain (MSVC 6.0 `CL.EXE` via wibo, decompnet)
* `_meta/version-differences.md` — global GC ↔ PS2 ↔ PC differences
* `Action.c/` — (module documentation in progress)
