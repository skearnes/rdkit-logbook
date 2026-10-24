# 2026-10-24 — Rust bindings for RDKit: design

- **Date:** 2026-10-24
- **Author:** Steven Kearnes
- **Acknowledgments:** Prepared with [Claude Code](https://claude.com/claude-code) (Claude Opus 5.5)
- **Status:** draft (design)
- **Tags:** rdkit, rust, bindings, cxx, conda-forge, design
- **License:** [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)

## Summary

Design for a Rust crate that binds RDKit's C++ API with the coverage of the
Python wrappers, so Rust users stop writing their own bindings. It starts as the
author's own crate, built against conda-forge's RDKit, and is laid out to move
into `rdkit/rdkit` once its patterns are settled. Decisions:

- It binds RDKit's C++ and reimplements nothing.
- The first release covers everything RDKit's nanobind Python wrappers expose.
  They, not the Boost.Python wrappers, are the reference.
- The API mirrors RDKit the way the nanobind bindings do, in Rust spelling.
- Users get RDKit from conda-forge, and the crate compiles its wrapper code
  against that install.
- Wrapper code is written by hand, by agents, following patterns settled on a
  core slice before the rest is written.

Follows [Reimplementing RDKit in
Rust](../2026-07-18-rdkit-rust-reimplementation/README.md), whose verdict on
full parity was to use the existing `rdkit-sys` bindings instead of porting. The
survey below finds those bindings stalled. Wrapper counts are against upstream
RDKit `master` at `7f3d37d` (2026-10-16); crate and repository state was read
from crates.io and GitHub on 2026-10-24.

## Why

Python users get two maintained bindings (Boost.Python and nanobind) and Java
users get SWIG. Rust users choose between third-party crates that cover a small
subset and writing their own.

| Crate | Approach | State |
| --- | --- | --- |
| `rdkit-sys`, `rdkit` ([rdkit-rs/rdkit](https://github.com/rdkit-rs/rdkit)) | cxx, one hand-written C++ function per call. Covers SMILES and molblock I/O, atoms, `RWMol` editing, fingerprints, descriptors, MolStandardize, substructure matching, scaffold networks, the periodic table. | The most used (about 55k downloads). Last release 0.4.12 on 2024-10-14, for RDKit 2024.09; last merge 2024-12-09. Seven open PRs, five from March 2026, one from the original author on 2026-10-09. |
| `rdkit-ffi`, `rdkit-rs` ([rogerwq/rdkit-rust](https://github.com/rogerwq/rdkit-rust)) | cxx, builds RDKit from a git submodule | Last release May 2022 |
| `rdk`, `rdk-sys` ([jcathalina/rdk-rs](https://github.com/jcathalina/rdk-rs)) | cxx, conda only | Last release April 2023 |

The main downstream user is [cheminee](https://github.com/rdkit-rs/cheminee), a
search engine built on `rdkit-sys`. The most active Rust cheminformatics crates
are reimplementations (`chematic`, `COSMolKit`, `sci-form`), which this design
does not compete with: it gives Rust users RDKit itself.

The existing bindings stall for three reasons, and the design answers each:

- **Coverage.** Each RDKit call needs its own C++ wrapper function, and each
  crate stopped at a small subset. Agents make writing those functions cheap, so
  the limit that remains is keeping them consistent.
- **Building.** Every `build.rs` guesses where RDKit is: Homebrew, conda, or
  `/usr/local`. On Linux, the `rdkit` README points at static RDKit tarballs the
  project hosts on S3.
- **Soundness.** `rdkit-sys` hands out `Pin<&mut Atom>` from a
  `&mut SharedPtr<ROMol>`, so two clones of one molecule pointer can produce
  aliasing mutable references in safe Rust. The open PR from 2026-10-09 is
  titled "Make CXX molecule ownership exclusive".

## Goals

- Every class, function, and method the nanobind Python modules expose has a
  Rust counterpart, or a recorded reason it has none.
- A user needs an activated conda environment with RDKit's development packages,
  and `cargo add`. Nothing else is installed or configured by hand, beyond one
  documented line that puts an rpath in their own binaries (see Constraints).
- User code needs no `unsafe`. The borrow checker enforces what nanobind's
  `keep_alive` enforces at run time.
- The repository moves into `rdkit/rdkit` without being restructured.

## Non-goals

- Reimplementing any RDKit algorithm in Rust.
- The pure-Python parts of the `rdkit` package, such as `PandasTools` and the
  `rdkit.Chem.Draw` helpers. Scope is what the compiled nanobind modules
  expose. *(My call, reading "what the Python wrappers cover" as the wrappers.)*
- A redesigned, idiomatic Rust API. The redesign is `rdkit.v3`, whose
  [molecule API](../2026-10-17-rdkit-v3-mol/README.md) is shaped for mechanical
  binding; binding it is later work.
- RDKit installs other than conda-forge's, for users. An override that points
  the build at a development build of RDKit is in scope, because developing the
  crate needs it.
- Prebuilt RDKit binaries, or building RDKit from source inside `cargo build`.
- Releases before parity.
- Passing molecules between Python and Rust in one process. In a conda
  environment Python's `rdkit` loads the same shared libraries, so the design
  should not preclude it.

## Anti-goals

Any of these makes the crate a failure even if it works on the author's machine:

- A build that finds RDKit by guessing paths.
- Safe Rust that can produce undefined behavior: a dangling atom reference,
  aliasing mutable references, a double free, a data race.
- A C++ exception that crosses into Rust uncaught and aborts the process.
- Wrapper code compiled with different preprocessor definitions from conda's
  RDKit. Some definitions change class layouts, so the mismatch corrupts memory
  rather than failing to compile.
- Coverage that stalls at a subset, as every predecessor's did.

## Constraints

- **conda-forge packages.** `librdkit` holds only the shared libraries.
  Headers and `lib/cmake/rdkit/rdkit-config.cmake` are in `librdkit-dev`. Boost
  headers come from `libboost-devel`, and compilers from `cxx-compiler`. The
  feedstock is at 2026.09.1.
- **conda-forge build options.** The feedstock turns on Avalon, Cairo, Contrib,
  FreeSASA, InChI, PostgreSQL, xyz2mol, and YAeHMOP, and turns off static
  libraries. Linking is dynamic, and a module conda-forge does not build cannot
  be bound.
- **Compilers.** Wrapper code compiles with conda's compilers and C++ standard
  library, not the system's, or binaries can fail at load time with missing
  symbols. CMake honors the `CXX` that conda activation sets.
- **Preprocessor definitions.** RDKit attaches its definitions to the
  `rdkit_base` interface target (`RDK_64BIT_BUILD`, `RDK_BUILD_THREADSAFE_SSS`,
  `RDK_USE_BOOST_SERIALIZATION`, `RDK_HAS_EIGEN3`, `RDK_BUILD_DESCRIPTORS3D`, and
  Boost settings). The build reads them from the installed CMake config; it
  does not hardcode them.
- **Finding shared libraries at run time.** conda does not set
  `LD_LIBRARY_PATH`. Cargo drops `rustc-link-search` paths outside the target
  directory from the library path it sets for `cargo run` and `cargo test`, and
  `rustc-link-arg` from the crate's build script reaches only the crate's own
  targets. Downstream binaries therefore need an rpath their own build adds, and
  the crate supplies the one-line way to add it.
- **Smart pointers.** RDKit returns `boost::shared_ptr<ROMol>` (`ROMOL_SPTR`)
  and vectors of them in many places. cxx supports only the standard library's
  smart pointers, so the C++ wrapper layer converts.
- **Version coupling.** Wrapper code compiles against RDKit's headers, so each
  crate release supports one RDKit release series. The build checks RDKit's
  version first and fails with a message naming the supported series. The
  crate's major and minor versions follow the series (2026.9.x supports RDKit
  2026.09), and development tracks the latest series on conda-forge. *(My
  call.)*
- **Names.** `rdkit` and `rdkit-sys` on crates.io belong to rdkit-rs.

## Decisions

### Where it lives

The author's own repository and crate, pinned to RDKit releases. The C++ wrapper
code sits in a directory CMake can build, so it can move under `Code/` later.
Moving in is proposed once the core slice's patterns hold and coverage is real.
The license is BSD-3-Clause, matching RDKit. *(License is my call.)*

### How users get RDKit

From conda-forge: an activated environment with `librdkit-dev`,
`libboost-devel`, `cxx-compiler`, and `cmake`. The build finds RDKit through
`CONDA_PREFIX`. One environment variable overrides the location, for
development builds of RDKit.

### Platforms

The first release supports Linux on x86_64 and aarch64 and macOS on arm64, and
CI builds and tests each. Windows comes later: conda-forge builds RDKit for it,
but its conda layout (`Library/`) and MSVC need their own build-script and CI
work.

### How Rust calls C++

cxx, with a hand-written C++ function wherever cxx cannot express the RDKit
signature directly: default arguments, Boost types, templates, raw owning
pointers, and exceptions that carry more than a message. *(My call. cxx is what
`rdkit-sys` uses, and with conda supplying headers there is no reason to drop
to a plain C layer.)*

The C++ side builds with CMake, not `cc`. *(My call.)* The build script
generates cxx's C++ with the `cxx-gen` crate, then runs CMake on the wrapper
directory. That project calls `find_package(rdkit)` on the installed config,
which supplies include paths, libraries, and RDKit's preprocessor definitions
without hardcoding them, and builds one static library the crate links. Moving
in-tree swaps `find_package` for RDKit's own targets; the directory stays as it
is.

### What the API looks like

It mirrors RDKit as the nanobind Python bindings do:

- The same classes (`ROMol`, `RWMol`, `Atom`, `Bond`, `Conformer`, and so on)
  and the same functions in snake_case (`mol_from_smiles`, `get_atom_with_idx`).
- Functions are grouped the way the Python modules group them, so Python code
  ports line by line. Where the nanobind and C++ shapes differ (a C++ free
  function bound as a Python method, logic that lives only in the nanobind
  wrapper), the nanobind shape wins and the C++ wrapper layer reproduces that
  logic. *(My call.)*
- Default and keyword arguments become parameter structs implementing
  `Default`, which RDKit's C++ is already moving toward.
- Overloads become separately named functions, under one naming rule applied
  everywhere.
- Exceptions become `Result` with an RDKit error type. Sanitization errors keep
  their atom indices. A parse that returns `None` in Python returns an error.
- Atoms and bonds are views that borrow their molecule. Editing takes a
  mutable reference.

### Scope and parity

The first release covers RDKit's nanobind modules: 64 modules, about 237
classes, and about 2,520 `def` calls (methods and free functions, overloads
counted separately), by a rough count of the `nbWrap` sources. Every
Boost.Python module has a nanobind counterpart at that commit.

Parity is measured, not estimated. *(My call.)* conda-forge's `rdkit` is built
with Boost.Python, so the inventory comes from an RDKit build with
`RDK_BUILD_NANOBIND_WRAPPERS` at the supported release. A script introspects
its compiled modules and writes every class, method, and function to an
inventory, regenerated once per RDKit release and checked in, so CI never
builds RDKit to check parity. A checked-in manifest maps each entry to a Rust
path, or to a reason
it has none (for example, Python-only protocol methods such as `__repr__` or
pickling hooks). APIs RDKit has deprecated are mapped to that reason rather than
bound. *(My call.)* CI fails on an unmapped entry. Each module's Python wrapper
tests are ported to Rust as that module is bound.

### Build order

Nothing ships before parity, but the core slice is built first, because every
later wrapper copies its patterns: molecules, atoms, bonds, conformers, ring
info, properties, SMILES, SMARTS, and molfile I/O, sanitization, and substructure
matching. The slice is done when each pattern below has one settled answer that
the rest of the crate copies:

- Owned molecules, borrowed atom and bond views, and mutation.
- How `RWMol` relates to `ROMol`, and in-place operations on a molecule.
- Functions that return new molecules, including vectors of `ROMOL_SPTR`.
- Parameter structs and defaults.
- The overload naming rule.
- Errors.
- Iterators over atoms, bonds, neighbors, and matches.
- Typed properties on molecules, atoms, and bonds.
- `Send` and `Sync`.

## Left to the builder

- **Thread safety.** Whether molecules are `Send`, `Sync`, both, or neither.
  RDKit computes some data lazily on `const` molecules, so read-only access from
  two threads is not automatically safe. Users will expect to parallelize over
  molecules, so the answer and its evidence go in the docs.
- **Errors.** cxx turns a caught exception into a message only. Keeping typed
  fields such as a sanitization error's atom indices needs its own mechanism.
- **`RWMol` and `ROMol`.** `Deref`, a trait, or separate types with conversions.
- **Crate split.** One crate, or a `-sys` crate under a safe one.
- **Module paths and the overload naming rule**, within the decisions above.
- **Python-specific types.** What NumPy arrays, file-like objects, callables,
  and dictionaries in Python signatures become in Rust (slices, `std::io::Read`,
  closures, maps), and how C++ streams read from Rust.
- **Optional modules.** Whether Avalon, FreeSASA, InChI, YAeHMOP, and the
  Cairo drawer sit behind Cargo features.
- **Test data.** The ported Python tests read files from RDKit's source tree,
  which conda does not ship; CI needs the matching release's sources.
- **Python interop.** What keeps in-process sharing with Python possible, such
  as `unsafe` conversions to and from raw `ROMol` pointers.
- **CI.** A conda environment per platform, built from scratch, so the build
  never passes because of something cached on a runner.
- **docs.rs.** Its builders have no conda, so documentation has to build
  without compiling or linking C++.

## Open questions

- The crate's name, given that `rdkit` and `rdkit-sys` are taken. This blocks
  publishing, not building.

## Next steps

- Settle the open questions.
- Plan the core slice: repository layout, build script, CI, and the patterns
  listed under Build order.
- Write the inventory script early, since it defines what done means.
