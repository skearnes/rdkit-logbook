# 2026-10-26 — Rust bindings for RDKit: report

- **Date:** 2026-10-26
- **Author:** Steven Kearnes
- **Acknowledgments:** Prepared with [Claude Code](https://claude.com/claude-code)
- **Status:** final (report; release pending)
- **Tags:** rdkit, rust, bindings, cxx, safety, threads, report
- **License:** [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)

## Summary

The crate from the [design entry](../2026-10-24-rdkit-rust-bindings/README.md)
now exists: [skearnes/rdkit-cxx](https://github.com/skearnes/rdkit-cxx) binds
RDKit's C++ API with the coverage of its nanobind Python wrappers. Every one of
the 3,910 names those wrappers expose in RDKit 2026.09.1 has a Rust counterpart
or a recorded reason it has none. It took 46 pull requests, from 2026-10-24 to
2026-10-26, following the [core slice plan](../2026-10-24-rdkit-rust-bindings/assets/CORE_SLICE_PLAN.md)
and then binding module by module.

The design's hardest goal was that no safe Rust can reach undefined behavior,
even where RDKit's Python wrappers can. Holding to it meant checking much of
what RDKit trusts, locking the global state it writes without locks, and
leaving 52 names unbound. Two gaps remain that the wrappers cannot close
cheaply (see [Known limitations](#known-limitations)). Along the way the work
turned up dozens of defects in RDKit itself, listed in
[RDKit bugs found](assets/RDKIT_BUGS.md); none has been reported upstream yet.

The crate also ports much of the functionality RDKit currently offers only
in Python, measured the same way. Of the 2,605 names RDKit's Python modules
define, 1,017 are ported, and the ports compute what Python computes. Each of
the rest has a recorded reason: Python-ecosystem glue, legacy code whose
replacement is bound, deprecated code, command-line plumbing, or Python
protocol hooks
(see [Ports of Python-only functionality](#ports-of-python-only-functionality)).
Five sessions ported them in parallel, and the ports found bugs in RDKit's
Python modules too.

The crate is not yet released. The first release, 2026.9.0, is ready in a
pull request and waits on running the release workflow; the `rdkit-cxx` name
on crates.io is held by a 0.0.0 placeholder.

## What exists

| | |
| --- | --- |
| Inventory | 3,910 names in RDKit's 64 nanobind modules (1,655 methods, 756 functions, 626 properties, 383 enum and flag values, 309 classes, 181 module attributes) |
| Bound | 3,478 |
| Recorded reasons | 432: 355 Python-only (protocol methods, nanobind's container types, Python callbacks), 52 unbound for safety, 13 deprecated, 12 that serve only unbound names |
| Python modules | 2,605 names in RDKit's Python (`.py`) modules: 1,017 ported; 150 with a reason of their own; 1,438 through 35 module reasons (`sping` alone is 726) |
| Code | About 86,000 lines of Rust in `src/`, 45,000 of C++ in `wrapper/` (not counting vendored headers), and 68,000 of tests in 156 files |
| Tests | 1,292 tests in 158 targets, many ported from RDKit's Python tests and checked against its Python output |
| CI | Linux x86-64, Linux arm64, and macOS arm64; a race check under Valgrind's DRD; pre-commit; docs built without RDKit; packaging |

The manifest that maps each name is checked in CI: an unmapped name fails the
build, and a generated test names every mapped Rust path, so `cargo test`
fails if one does not exist.

## How the open questions were settled

The design left these to the builder:

- **Name.** `rdkit-cxx`, mirroring RDKit's Python wrappers in Rust spelling.
- **Crate split.** One crate. The cxx bridges are private modules under
  `src/ffi/`, one per RDKit area, beneath a public API whose module paths follow
  Python's (`rdkit.Chem.rdmolfiles.MolFromSmiles` is
  `rdkit_cxx::chem::rdmolfiles::mol_from_smiles`).
- **`RWMol` and `ROMol`.** Every Rust `Mol` owns a C++ `RWMol`, so in-place
  operations need no conversion. `RWMol` wraps a `Mol` and derefs to it. Atoms,
  bonds, and conformers are views that borrow their molecule; a mutable view
  holds an index, so two can never alias.
- **Errors.** Every wrapper runs under a `trycatch` that encodes the
  exception's type and payload, such as a sanitization error's atom index, into
  the message cxx carries. The Rust side decodes it into an `Error` variant.
- **Overloads and defaults.** Optional arguments become `_with_` functions, or
  a params struct implementing `Default` when there are several.
- **Python-specific types.** Slices and `Vec`s for sequences, `Read` and
  `Write` for file-like objects, `BTreeMap`s for dictionaries, and Rust closures
  for Python callables. A closure crosses into C++ as an `extern "Rust"` type.
  A panic in one is caught at the boundary, and where RDKit could not survive
  an exception, as in FMCS's search, it is held until RDKit returns.
- **Thread safety.** `Mol` is `Send` and not `Sync`, since RDKit updates
  cached data inside `const` molecules. Global state RDKit writes without a
  lock gets a mutex in the wrapper, taken in one fixed order (see below).
- **Optional modules.** No Cargo features. conda-forge builds every optional
  module, so all of them are bound.
- **Test data.** The files the ported tests need are copied from RDKit's tree
  into `tests/data/`.
- **docs.rs.** With `DOCS_RS` set, the build script skips the C++, so the docs
  build without RDKit.
- **Python interop.** Not built: there are no conversions to or from raw
  `ROMol` pointers.

## What "no undefined behavior from safe Rust" took

RDKit trusts its callers in many places that Python's wrappers pass straight
through. The crate's rules (in its `CONVENTIONS.md`) handle them in a few ways:

- **Check what RDKit trusts.** A C++ wrapper stands between every bridge
  function and RDKit. Where RDKit trusts an argument (an index, a size it
  computes in `int`, a value it casts, a pickle's lengths, a file's line
  counts), the wrapper checks it first and returns an error. Each check carries
  a comment naming what RDKit would do without it.
- **Check binary formats, or make their readers `unsafe`.** RDKit's own binary
  formats (bit-vector, grid, and catalog pickles, FPB files, PNG chunks) are
  checked before RDKit reads them. Boost.Serialization archives cannot be
  checked, so their readers are `unsafe fn`s whose contract is "bytes RDKit
  wrote". There are 16 `unsafe fn`s in all. The rest are parsers that write
  the SMILES parser's unlocked debug level, the synthon-space database reader,
  and setters for settings RDKit keeps in environment variables.
- **Lock global state.** RDKit writes some process-wide state with no lock:
  the random number generator, 2D depiction settings, the BRICS patterns, the
  ChemDraw library's attribute maps, Murtagh clustering's statics, and others.
  Most importantly, every substructure match swaps the `SIGINT` handler through
  static variables. Each wrapper that reaches such state holds a mutex for the
  whole RDKit call. CI runs a thread phase for each lock under Valgrind's DRD,
  so a missing lock fails the build. CI also turned up a real race in the
  ChemDraw library, which DRD then confirmed.
- **Check before removals.** Whenever RDKit removes atoms or bonds, it follows
  substance-group `PARENT` chains with no check for cycles and parses
  `_MolFileBondEndPts` unchecked. The SMILES writer removes atoms from copies to
  split fragments, so this covers nearly everything that writes SMILES.
- **Leave it unbound.** Where RDKit's code cannot be made safe from outside,
  the name is left unbound, with the reason and the crash that confirmed it in
  the manifest: the Avalon toolkit, `rdEHTTools`, `MaeMolSupplier`, InChI's
  mol-file reader, the ChemDraw readers, and the synthon space's shape search.

The cost is that some inputs Python accepts are refused, such as a parent
hierarchy across molecules that RDKit would handle, or an InChI option outside
the structure-perception set. Substructure matching is also serialized across
the whole process by the Ctrl-C lock.

## Known limitations

Safe Rust can still reach undefined behavior in two ways. Neither corrupts
memory on the supported platforms, and neither can be closed in the wrappers
without disproportionate cost:

- **RDKit's log flags race.** Each logger's enabled flag is a plain `bool` that
  every log statement reads. RDKit writes it inside common calls (`hasQueryHs`
  in `AddHs.cpp`, drawing, deprotection, R-group decomposition),
  and `BlockLogs` and `disable_log` write it too. Two threads using RDKit at
  once can race on it. The crate orders RDKit's own writers with each other,
  but closing the race would mean locking every RDKit call. The fix belongs
  upstream: make the flag a `std::atomic<bool>`.
- **A float cast in molecule pickles.** `MolPickler` computes an isotope as
  `static_cast<int>(floor(tmpFloat + mass))` from a float in the pickle, which
  is undefined for a value that does not fit in `int`. Checking it from outside
  would mean parsing the pickle format in the wrapper. The fix belongs
  upstream: a range check before the cast.

Other open items:

- **Stack overflows.** Some inputs overflow the stack, which aborts the
  process but cannot corrupt memory: an SLN string with a `{` followed by
  about 100,000 characters (RDKit's regex), and SMILES of very deep molecules
  on small thread stacks.
- **Fixed before release.** The extended-query and R-group decomposition
  paths now run the crate's enumeration checks before RDKit's enumerator,
  including its limit of 4,000 variation points, and `MultithreadedMolSupplier`
  starts at most 1,024 writer threads.
- **Windows.** Unsupported: conda-forge's layout and MSVC need their own
  build-script and CI work, and the clustering bindings assume a 64-bit
  `long`.

## Ports of Python-only functionality

The design left RDKit's Python modules out of scope. They were ported after
parity so that the first release covers much of what RDKit offers from
Python, not only the compiled modules:

- **Measured like the bindings.** `inventory.py --python` reads each `.py`
  module under `rdkit/` (without its tests) and lists the public names it
  defines, 2,605 in all. Each gets a manifest entry, ported or with a reason.
  Glue and legacy code takes one reason for its whole module, and
  `check.py --strict` fails on any module that is neither complete nor
  covered by a reason.
- **Ported on the bindings.** Each port calls the bound API and computes
  what Python computes. Where a result depends on Python itself, the crate
  reproduces it: Python's Mersenne Twister and its seeding, draw for draw;
  NumPy's pairwise summation; the shortest `repr` of a float. The modules'
  Python tests and doctests are ported with them, with expected values from
  RDKit 2026.09.1.
- **In parallel.** Five cloud sessions each owned one group of modules:
  descriptors; chemistry utilities (BRICS, Recap, MolKey, scaffolds,
  enumeration); fingerprints and pharmacophores; `Draw`; and machine
  learning and data structures. They worked in seven pull requests and
  changed the shared manifest only through `manifest_add.py`, which merges
  entries into it, so their pull requests merged without losing each
  other's entries.
- **Not ported.** Python-ecosystem glue (`PandasTools`, `IPythonConsole`,
  PyMOL and Windows COM drivers), legacy code whose replacement is bound
  (`sping`, the old `MolDrawing`, the MCS search in `fmcs`), and code RDKit
  deprecates (the `Chem.Fingerprints` package, `PropertyMol`,
  `FastSDMolSupplier`), each with a reason that names the replacement.

Several Python functions are broken in RDKit 2026.09.1, mostly under NumPy 2
or Python 3. The ports implement what they intend, and
[RDKit bugs found](assets/RDKIT_BUGS.md#rdkits-python-modules) lists them.

## Release status

The first release, 2026.9.0, documents the two known limitations rather than
wait for upstream fixes. Its pull request
([rdkit-cxx#62](https://github.com/skearnes/rdkit-cxx/pull/62)) removes
`publish = false`, describes the ports in the crate's description and README,
and adds a changelog. What remains:

- The release workflow (run by hand) tests the release commit, publishes
  through crates.io's trusted publishing, and tags the release. It has not been
  run.
- The packaged crate is 1.7 MiB.

## Next steps

- Run the release workflow for 2026.9.0.
- Report the RDKit bugs upstream, starting with the memory-safety ones, and
  open the drafted Ctrl-C fix (`skearnes/rdkit`, branch
  `thread-safe-controlc`). Report the Python modules' bugs as well.
- Ask upstream for atomic log flags and a checked cast in `MolPickler`, then
  drop the two limitations.
- Windows support.
