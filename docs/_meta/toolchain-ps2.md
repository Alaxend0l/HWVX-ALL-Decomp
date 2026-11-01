# PS2 toolchain — MW MIPS C Compiler 2.4.1.01 (CodeWarrior PS2 R3.01)

## What the binary tells us

* `HWVX-PS2-Decomp/game.elf` banner string: `MW MIPS C Compiler (2.4.1.01)` — the
  frontend version the retail PS2 build was compiled with.
* The retail ELF carries CodeWarrior-PS2-style GCC-extension sections:
  `.gcc_except_table`, `.lit8`, `.lit4` (float/literal pools), plus the usual
  `.text/.data/.rodata/.sdata/.sbss/.bss`. `.vutext/.vudata/.vubss` are
  reserved in the splat config but unused by this binary (no VU micro-code
  sections present).
* `_gp = 0x002D7EF0` — small-data area addressed through the gp register, so
  the decomp must keep small data (`_s*` symbols) in `.sdata/.sbss`.
* Code segment VRAM base is `0x00200000` (ELF loaded at 0x200000); file size to
  end of code ≈ `0xD0080`.

## Available toolchain: CodeWarrior for PS2 R3.01 (CLT)

Located in the repo at `tools/CodeWarrior_for_PS2_3.01/` (`CW_PS2_R3.01.exe`
self-extractor + release notes; git-ignored). Extracted to
`/home/alaxendol/Decomp/.cw_extract/clt/Command_Line_Tools/` containing the
Windows PE32 tools:

| Tool | Purpose | Version when run |
| --- | --- | --- |
| `mwccps2.exe` | C/C++ compiler (MIPS/PS2) | runtime 3.0.1; banner `MW MIPS C Compiler (2.4.1.01)` |
| `mwldps2.exe` | linker | 3.0.1 |
| `asm_r5900_elf.exe` | R5900 assembler | — |

They run on this Linux host under Wine (wine-staging 11.20, `WINEPREFIX=
/home/alaxendol/Decomp/.cw_extract/wine`). Verified working:
`wine mwccps2.exe -v` prints `Metrowerks C/C++ Compiler for MIPS/PlayStation2 … Version 3.0.1`.

Notes:
* `mwccps2` needs `--gcc-compat` as the **first** option (or env
  `MWCL_GCC_COMPAT`) to accept GCC-style flags; help was captured to
  `mwccps2_help_gcc.txt`.
* Per the bundled `PS2_setup.txt`, the intended setup is:
  * `MWCIncludes` = GCC library headers + "PS2 Support" folder
  * `MWLibraries` = GCC library dirs; `MWLibraryFiles` = `;`-separated list
  * a license file next to the CLT binaries
  * the CW-provided custom LCF from the "PS2 Support" folder passed to the linker
  * `gcc_wrapper.c` may be required
  * entry point via linker option `-m ENTRYPOINT`

## What the decomp harness expects

`game.elf.yaml` (splat9, `compiler: MWCCPS2`) currently builds a **single**
`src/cod.c` → `build/src/cod.o` and links with GNU-ld-syntax
`game.elf.ld` (+ optional `linker_script_extra.ld`, `ENTRY(_start)`),
producing `build/game.elf.elf` to be byte-compared against
`game.elf.rom` (sha1 `1b525c7434d54651203f3a75c2cd756dfc1a33ee`).
A leftover depfile `build/src/cod.asmproc.d` shows decompdecomp's asm-processing
step ran at least once.

Progress reference (`progress/README.md` + `progress/decompdecomp.json`): best
scoring attempts used `mwcps2-3.0.1 -O2 -sdatathreshold 8` — i.e. the same
CodeWarrior R3.01 compiler. `progress/best/` holds 774 per-function best C
attempts (m2c machine drafts, `M2C_*` macros), `progress/src/` 33 named-module
attempts (e.g. `Action_Destroy.c`).

## Open setup work

1. Split `src/cod.c` into GC-mirrored modules (splat `subsegments` +
   `find_file_boundaries` once functions are named in `symbol_addrs.txt`).
2. Provide a build entry point (makefile or reuse decompdecomp harness) that
   compiles `src/*.c` with `mwccps2 --gcc-compat -O2 -sdatathreshold 8` (Wine)
   and links `build/game.elf.elf`, then `compare`.
