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

The crate is not yet released. The first release will also port the
pure-Python parts of the `rdkit` package that make sense in Rust, so that it
covers most of what RDKit offers from Python, not only the compiled modules.
The release workflow is ready, and the `rdkit-cxx` name on crates.io is held
by a 0.0.0 placeholder.

## What exists

| | |
| --- | --- |
| Inventory | 3,910 names in RDKit's 64 nanobind modules (1,655 methods, 756 functions, 626 properties, 383 enum and flag values, 309 classes, 181 module attributes) |
| Bound | 3,478 |
| Recorded reasons | 432: 355 Python-only (protocol methods, nanobind's container types, Python callbacks), 52 unbound for safety, 13 deprecated, 12 that serve only unbound names |
| Code | About 55,000 lines of Rust in `src/`, 45,000 of C++ in `wrapper/` (not counting vendored headers), and 39,000 of tests in 103 files |
| Tests | 971 tests in 106 targets, many ported from RDKit's Python tests and checked against its Python output |
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
- **To fix before release.** The extended-query and R-group decomposition
  paths call RDKit's enumerator without the crate's enumeration checks,
  including its limit of 4,000 variation points, and `MultithreadedMolSupplier`
  can start up to `INT_MAX` writer threads.
- **Windows.** Unsupported: conda-forge's layout and MSVC need their own
  build-script and CI work, and the clustering bindings assume a 64-bit
  `long`.

## Release status

The first release, 2026.9.0, waits for the pure-Python ports and the two fixes
above. It will document the two known limitations rather than wait for
upstream fixes. The mechanics are ready:

- `Cargo.toml` still sets `publish = false`, which the release workflow's dry
  run refuses, and the README's status line predates parity.
- The release workflow (run by hand) tests the release commit, publishes
  through crates.io's trusted publishing, and tags the release. It has not been
  run.
- The packaged crate is 1.2 MiB.

## Next steps

- Port the pure-Python parts of the `rdkit` package, which the design left out
  of scope. Measure them with the same inventory and manifest, so each public
  name gets a port or a recorded reason. Code that is Python-ecosystem glue
  (`PandasTools`, `IPythonConsole`) or legacy (`sping`, `Dbase`) gets reasons.
  The first ports have landed with the RDKit Cookbook's examples:
  `RegistrationHash`, `Lipinski`, `FindMolChiralCenters`, and the parts of
  `Draw` the recipes use.
- Fix the enumeration checks and the supplier's thread count, then remove
  `publish = false`, update the README, document the known limitations, and
  run the release workflow.
- Report the RDKit bugs upstream, starting with the memory-safety ones, and
  open the drafted Ctrl-C fix (`skearnes/rdkit`, branch
  `thread-safe-controlc`).
- Ask upstream for atomic log flags and a checked cast in `MolPickler`, then
  drop the two limitations.
- Windows support.
