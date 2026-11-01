# GC toolchain — CodeWarrior GC 1.3.2 (reference build)

The GC decomp (`HWVX-GC-Decomp`) is the complete reference and defines the
canonical file/function names used by PS2 and PC docs.

* Game: *Hot Wheels Velocity X* — banner `HOTWHE`, `main.dol` in `orig/HOTWHE/`.
* Harness: [dtk-template](https://github.com/encounter/dtk-template) driven by
  `configure.py` + `ninja`; every build rebuilds `main.dol` and byte-compares it.
* Started with [decompdecomp](https://github.com/n-delic/decompdecomp);
  progress shown per function via objdiff (`objdiff.json`).
* Compiler: CodeWarrior GC 1.3.2 (`mwccppc`), fetched by `configure.py`.
  Objects carry the CodeWarrior `.comment` section semantics
  (see `HWVX-GC-Decomp/docs/comment_section.md`).
* Layout: 309 split files under `src/` (plus `include/`), one `.c` per original
  translation unit; `assets/` and `dtk/` support data/symbol handling.
