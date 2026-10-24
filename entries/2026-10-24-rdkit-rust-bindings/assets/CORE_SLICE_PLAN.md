# Rust bindings for RDKit: core slice implementation plan

- **Date:** 2026-10-24
- **Author:** Steven Kearnes
- **Acknowledgments:** Prepared with [Claude Code](https://claude.com/claude-code) (Claude Opus 5.5)
- **License:** [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> superpowers:executing-plans (recommended, since the author prefers work done
> in process) or superpowers:subagent-driven-development to implement this
> plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build the repository, the build pipeline, the parity tooling, and
the core slice of RDKit bindings whose patterns every later wrapper copies.

**Architecture:** One Cargo crate. Its build script generates cxx's C++ with
`cxx-gen`, then builds that C++ and the hand-written wrapper functions with
CMake against conda-forge's RDKit (`find_package(rdkit)`), and links the
result. Private cxx bridges under `src/ffi/` sit beneath a public API that
mirrors the nanobind Python modules. An inventory of the nanobind modules and a
checked-in manifest measure parity.

**Tech Stack:** Rust 2024 edition; `cxx` 1.0.205 and `cxx-gen` 0.7.205; the
`cmake` crate; CMake 3.22 or later; C++20; conda-forge `librdkit-dev` 2026.09;
Python 3.12 with pytest for the inventory tooling; GitHub Actions.

**Spec:** [the design entry](../README.md). Executors read both documents.

## Global Constraints

- Bind RDKit's C++; reimplement nothing.
- The reference surface is RDKit's nanobind Python wrappers at tag
  `Release_2026_09_1`, not the Boost.Python wrappers.
- RDKit comes from conda-forge (`librdkit-dev`, `libboost-devel`,
  `cxx-compiler`, `cmake`), release series 2026.09, linked dynamically.
- `RDKIT_PREFIX`, when set, overrides `CONDA_PREFIX` as the RDKit install
  prefix. No other variable selects RDKit.
- Crate version 2026.9.x supports RDKit 2026.09. The build derives the
  supported series (`2026.09`) from `CARGO_PKG_VERSION_MAJOR` and
  `CARGO_PKG_VERSION_MINOR`, and nowhere else.
- Platforms: Linux x86_64, Linux aarch64, macOS arm64.
- License: BSD-3-Clause.
- User code needs no `unsafe`.
- C++ builds with CMake through `find_package(rdkit)`, as C++20 (RDKit
  2026.09's standard). No RDKit path or preprocessor definition is hardcoded.
- `cxx = "=1.0.205"` and `cxx-gen = "=0.7.205"` are pinned and move together.
- The only runtime dependency is `cxx`. Build dependencies are `cxx-gen`,
  `proc-macro2`, and `cmake`. No dev-dependencies.
- Edition 2024, `rust-version = "1.85"`.
- The package and library are named `rdkit`, with `publish = false` until the
  crate's name is settled. The repository is `rdkit-rust`, at `~/rdkit/rdkit-rust`.
- `wrapper/` is a self-contained CMake project, so it can move under RDKit's
  `Code/` unchanged.
- Each rule a task adds to `CONVENTIONS.md` binds every later task.
- Every task ends with `cargo fmt --check`,
  `cargo clippy --workspace --all-targets -- -D warnings`, and
  `cargo test --workspace` passing, and from Task 3 on,
  `python3 inventory/check.py --write-paths tests/manifest_paths.rs` passing
  with no change to the generated file left uncommitted.
- Commit messages end with the `Co-Authored-By:` trailer for the model doing
  the work. The initial empty commit is the only commit made directly on
  `main`; all work happens on the `core-slice` branch.

## Review Focus

1. **Building outside an activated environment.** With neither `CONDA_PREFIX`
   nor `RDKIT_PREFIX` set, `cargo build` fails with one message naming the four
   conda packages and `RDKIT_PREFIX`, not a CMake trace. (Task 1 checks it
   locally, Task 2 in CI.)
2. **A downstream binary with no library path.** A crate that depends on this
   one, run with `LD_LIBRARY_PATH` and every `DYLD_*` variable unset, starts and
   finds conda's libraries through the one documented build-script line. (Task 2.)
3. **Molecules that skipped sanitization.** On a molecule built atom by atom or
   parsed with `sanitize: false`, calls that need a property cache or ring info
   return `Err(Error::Runtime)` instead of aborting, and substructure matching
   still works. (Tasks 6, 9, 11.)
4. **Text that is not UTF-8.** A molfile whose title line holds Latin-1 bytes
   loads; reading `_Name` or writing a molblock from it returns
   `Err(Error::Value)` (Python raises `UnicodeDecodeError`, a `ValueError`)
   instead of aborting. (Tasks 7, 8.)
5. **Empty inputs.** An empty SMILES string yields a molecule with no atoms that
   writes back as `""`; its iterators and fragment lists are empty; a molecule
   with no conformers returns `Err(Error::Value)` from `get_conformer`.
   (Tasks 4, 5, 9, 10.)

---

## Repository layout

```text
rdkit-rust/
  Cargo.toml  build.rs  LICENSE  README.md  CONVENTIONS.md  environment.yml
  .github/workflows/ci.yml
  wrapper/                     CMake project; moves under Code/ later
    CMakeLists.txt
    cmake/RDKitRustVersion.cmake
    include/rdkit-rust/<area>.h
    src/<area>.cpp
  src/
    lib.rs  error.rs  rd_base.rs
    ffi/mod.rs  ffi/<area>.rs    private cxx bridges, one per wrapper header
    geometry/  chem/             public API, mirroring Python's module paths
  tests/<topic>.rs  tests/manifest_paths.rs (generated)  tests/cmake/
  downstream/                  workspace member that depends on the crate
  inventory/                   inventory.py  check.py  manifest.toml  tests
```

Areas, in task order: `rd_base`, `errors`, `mol`, `molfiles`, `atom`, `bond`,
`props`, `conformer`, `ring_info`, `molops`, `substruct`.

## Tasks

### Task 1: Repository skeleton and build pipeline

**Files:**

- Create: `Cargo.toml`, `build.rs`, `LICENSE`, `README.md`, `CONVENTIONS.md`,
  `environment.yml`, `.gitignore`
- Create: `wrapper/CMakeLists.txt`, `wrapper/cmake/RDKitRustVersion.cmake`,
  `wrapper/include/rdkit-rust/rd_base.h`, `wrapper/src/rd_base.cpp`
- Create: `src/lib.rs`, `src/error.rs`, `src/ffi/mod.rs`, `src/ffi/rd_base.rs`,
  `src/rd_base.rs`
- Test: `tests/rd_base.rs`, `tests/cmake/version_check.cmake`

**Interfaces:**

- Produces:
  - `rdkit::rd_base::{rdkit_version() -> String, boost_version() -> String,
    disable_log(spec: &str), enable_log(spec: &str)}`, mirroring
    `rdBase.rdkitVersion`, `rdBase.boostVersion`, `rdBase.DisableLog`, and
    `rdBase.EnableLog`.
  - `pub(crate) fn expect_ok<T>(result: Result<T, cxx::Exception>) -> T` in
    `src/error.rs`, panicking with `unexpected RDKit exception: {what}`.
  - `const BRIDGES: &[&str]` in `build.rs`, one name per `src/ffi/<name>.rs`.
  - CMake lists `RDKIT_RUST_SOURCES` (wrapper `.cpp` files) and
    `RDKIT_RUST_RDKIT_TARGETS` (RDKit targets without the `RDKit::` prefix).
  - CMake functions `rdkit_rust_version_supported(<version> <series> <out>)` and
    `rdkit_rust_check_version(<version> <series>)`.
  - Build-script metadata `libdir`, which downstream build scripts read as
    `DEP_RDKIT_LIBDIR`.
  - Generated C++ under `$OUT_DIR/cxxgen`: `include/rust/cxx.h` (written from
    `cxx_gen::HEADER`), `include/rdkit-rust/ffi/<name>.rs.h`, and
    `src/<name>.rs.cc`.

- [ ] **Step 1: Create the repository and the environment**

  `git init -b main ~/rdkit/rdkit-rust`, commit an empty initial commit, and
  `git switch -c core-slice`. Write `environment.yml`:

  ```yaml
  name: rdkit-rust
  channels:
    - conda-forge
  dependencies:
    - librdkit-dev 2026.09.*
    - libboost-devel
    - cxx-compiler
    - cmake >=3.22
    - python 3.12.*
    - pytest
  ```

  Run: `mamba env create -f environment.yml && conda activate rdkit-rust &&
  ls "$CONDA_PREFIX/lib/cmake/rdkit/rdkit-config.cmake"`
  Expected: the file exists.

- [ ] **Step 2: Write the failing CMake test** in
  `tests/cmake/version_check.cmake`:

  ```cmake
  include(${CMAKE_CURRENT_LIST_DIR}/../../wrapper/cmake/RDKitRustVersion.cmake)
  foreach(case "2026.09.1:TRUE" "2026.09.2:TRUE" "2026.09.1b1:TRUE"
               "2026.03.4:FALSE" "2027.03.1:FALSE" "2026.9.1:FALSE")
    string(REPLACE ":" ";" case "${case}")
    list(GET case 0 version)
    list(GET case 1 expected)
    rdkit_rust_version_supported("${version}" "2026.09" ok)
    if(NOT ok STREQUAL expected)
      message(FATAL_ERROR "${version}: expected ${expected}, got ${ok}")
    endif()
  endforeach()
  message(STATUS "version check tests passed")
  ```

  Run: `cmake -P tests/cmake/version_check.cmake`
  Expected: FAIL, `RDKitRustVersion.cmake` not found.

- [ ] **Step 3: Implement `wrapper/cmake/RDKitRustVersion.cmake`.**
  `rdkit_rust_version_supported` sets `<out>` to `TRUE` exactly when
  `<version>` begins with `<series>.`, else `FALSE`.
  `rdkit_rust_check_version` stops on `FALSE` with:
  `This rdkit crate supports RDKit <series>.*, but found RDKit <version> at
  <rdkit_DIR>. Install librdkit-dev <series>.* or point RDKIT_PREFIX at a
  matching install.`

- [ ] **Step 4: Run the CMake test.**
  Expected: `version check tests passed`.

- [ ] **Step 5: Write the failing Rust test** in `tests/rd_base.rs`:

  ```rust
  use rdkit::rd_base;

  #[test]
  fn rdkit_version_is_the_supported_series() {
      assert!(rd_base::rdkit_version().starts_with("2026.09."));
  }

  #[test]
  fn boost_version_is_set() {
      assert!(!rd_base::boost_version().is_empty());
  }

  #[test]
  fn log_can_be_disabled_and_enabled() {
      rd_base::disable_log("rdApp.error");
      rd_base::enable_log("rdApp.error");
  }
  ```

- [ ] **Step 6: Write `Cargo.toml`** with `name = "rdkit"`,
  `version = "2026.9.0"`, `edition = "2024"`, `rust-version = "1.85"`,
  `license = "BSD-3-Clause"`, `links = "rdkit"`, `publish = false`, and the
  dependencies from Global Constraints (`cmake = "0.1"`, `proc-macro2 = "1"`).
  Run: `cargo test --test rd_base`
  Expected: FAIL, the package has no `src/lib.rs` yet.

- [ ] **Step 7: Write the C++ side.** `rd_base.h` declares, in namespace
  `rdkit_rust`: `std::unique_ptr<std::string> rdkit_version()`,
  `std::unique_ptr<std::string> boost_version()`,
  `void disable_log(rust::Str spec)`, and `void enable_log(rust::Str spec)`.
  `rd_base.cpp` returns `RDKit::rdkitVersion` and `RDKit::boostVersion` and
  makes the same logging calls as `EnableLog` and `DisableLog` in
  `Code/RDBoost/nbWrap/RDBase.cpp`.

- [ ] **Step 8: Write the bridge** `src/ffi/rd_base.rs`, which fixes the
  pattern later bridges copy:

  ```rust
  #[cxx::bridge(namespace = "rdkit_rust")]
  pub(crate) mod ffi {
      unsafe extern "C++" {
          include!("rdkit-rust/rd_base.h");
          fn rdkit_version() -> Result<UniquePtr<CxxString>>;
          fn boost_version() -> Result<UniquePtr<CxxString>>;
          fn disable_log(spec: &str) -> Result<()>;
          fn enable_log(spec: &str) -> Result<()>;
      }
  }
  ```

- [ ] **Step 9: Write `wrapper/CMakeLists.txt`.** `cmake_minimum_required`
  3.22, C++20 required, position-independent code on. It includes
  `cmake/RDKitRustVersion.cmake`, calls `find_package(rdkit CONFIG REQUIRED)`,
  then `rdkit_rust_check_version("${rdkit_VERSION}"
  "${RDKIT_RUST_SUPPORTED_SERIES}")`. It builds static library `rdkitrust` from
  `RDKIT_RUST_SOURCES` (here `src/rd_base.cpp`) plus
  `RDKIT_RUST_GENERATED_SOURCES`, with include directories `include`,
  `RDKIT_RUST_GENERATED_INCLUDE_DIR`, and `RDKit_INCLUDE_DIRS`. It links
  `RDKit::<target>` for each entry of `RDKIT_RUST_RDKIT_TARGETS` (here
  `RDGeneral`). It writes `${CMAKE_BINARY_DIR}/link-libraries.txt` with
  `file(GENERATE)`, one `$<TARGET_LINKER_FILE:RDKit::<target>>` per line. The
  build script passes `RDKIT_RUST_SUPPORTED_SERIES`,
  `RDKIT_RUST_GENERATED_SOURCES` (`;`-separated), and
  `RDKIT_RUST_GENERATED_INCLUDE_DIR`.

- [ ] **Step 10: Write `build.rs`**, in this order:
  1. Prefix: `RDKIT_PREFIX`, else `CONDA_PREFIX`, else panic with:
     `rdkit: no RDKit install found. Activate a conda environment with
     librdkit-dev, libboost-devel, cxx-compiler, and cmake, or set RDKIT_PREFIX
     to an RDKit install prefix.`
  2. `rerun-if-env-changed` for `RDKIT_PREFIX`, `CONDA_PREFIX`, `CXX`, and
     `DOCS_RS`; `rerun-if-changed` for `wrapper` and `src/ffi`.
  3. For each name in `BRIDGES`: read `src/ffi/<name>.rs`, parse it as a
     `proc_macro2::TokenStream`, call `cxx_gen::generate_header_and_cc` with
     `Opt::default()`, and write the header and implementation to the layout
     above. Write `cxx_gen::HEADER` to `include/rust/cxx.h`.
  4. `cmake::Config::new("wrapper")` with `CMAKE_PREFIX_PATH` set to the
     prefix, the three variables from Step 9, `.profile("Release")`, and
     `.build_target("rdkitrust")`.
  5. Link: search `<dst>/build` and link `static=rdkitrust`. For each line of
     `link-libraries.txt`, search its directory and link `dylib=<name>`, where
     `<name>` drops the `lib` prefix and everything from the first `.so` or
     `.dylib`. Link the C++ standard library (`c++` on macOS, `stdc++` on
     Linux), and when `$CXX -print-file-name=<libc++.dylib | libstdc++.so>`
     prints an absolute path, add its directory to the search path, so the link
     uses the library the wrappers compiled against. Emit
     `rustc-link-arg=-Wl,-rpath,<RDKit library directory>` for the crate's own
     tests and examples, and `cargo::metadata=libdir=<RDKit library directory>`.

- [ ] **Step 11: Write `src/lib.rs`** (`mod error; mod ffi; pub mod rd_base;`),
  `src/ffi/mod.rs`, and `src/rd_base.rs`. Each public function calls its bridge
  function through `expect_ok` and converts the `CxxString` with `to_str()`.
  Its doc comment names the Python object it mirrors.

- [ ] **Step 12: Run the tests.**
  Run: `cargo test --test rd_base`
  Expected: PASS, 3 tests.

- [ ] **Step 13: Check the missing-environment message.**
  Run: `env -u CONDA_PREFIX -u RDKIT_PREFIX cargo build 2>&1 | grep -F
  "or set RDKIT_PREFIX"`
  Expected: the Step 10 message, with no CMake output above it.

- [ ] **Step 14: Write the documents.** `LICENSE`: BSD-3-Clause, `Copyright
  (c) 2026, Steven Kearnes`. `README.md`: what the crate is, the environment,
  `cargo test`. `CONVENTIONS.md`, with these rules:
  - **B1.** Every bridge function is a free function in namespace
    `rdkit_rust`, declared in `wrapper/include/rdkit-rust/<area>.h` and defined
    in `wrapper/src/<area>.cpp`. Bridges never bind RDKit members directly.
  - **B2.** Every bridge function returns `Result`. A C++ exception that
    reaches cxx through a function without `Result` aborts the process.
  - **B3.** One bridge, `src/ffi/<area>.rs`, per wrapper header. An opaque
    type is declared in one bridge and reused in others as
    `type X = crate::ffi::<area>::ffi::X;`.
  - **B4.** A new area appends to `BRIDGES`, to `RDKIT_RUST_SOURCES`, and to
    `RDKIT_RUST_RDKIT_TARGETS` when it needs another RDKit library.
  - **T2.** C++ strings cross the bridge as `UniquePtr<CxxString>`, never
    `rust::String` (whose constructor throws on invalid UTF-8). Rust converts
    them with `to_str()`.
  - **N1.** Module paths mirror Python's, each component in snake_case:
    `rdkit.rdBase` → `rdkit::rd_base`, `rdkit.Chem.rdchem` →
    `rdkit::chem::rdchem`, `rdkit.Geometry.rdGeometry` →
    `rdkit::geometry::rd_geometry`. `rdkit::chem` re-exports its submodules
    the way `rdkit.Chem` does.
  - **N2.** Functions, methods, properties, and module attributes take the
    snake_case of the Python name (`snake_case` in `inventory/check.py`) and
    keep prefixes such as `get_`. Module attributes become zero-argument
    functions.
  - **N3.** Classes keep the Python class name: `Mol`, not `ROMol`.
  - **N4.** Enum values become UpperCamelCase variants (`BondType.SINGLE` →
    `BondType::Single`). Flag values keep Python's spelling as associated
    constants (`SanitizeFlags::SANITIZE_ALL`).
  - **N5.** Exceptions to N2–N4 are recorded in the manifest with
    `name_exception`.

- [ ] **Step 15: Commit**: `git add` the files above;
  `git commit -m "Build the wrapper with CMake against conda's RDKit"`.

### Task 2: CI, a downstream crate, and docs.rs

**Files:**

- Create: `.github/workflows/ci.yml`, `downstream/Cargo.toml`,
  `downstream/build.rs`, `downstream/src/main.rs`
- Modify: `Cargo.toml` (`[workspace] members = [".", "downstream"]`),
  `build.rs`, `README.md`

**Interfaces:**

- Consumes: `rdkit::rd_base::rdkit_version`, `DEP_RDKIT_LIBDIR`.
- Produces: binary `downstream`, which prints `RDKit <version>`; CI jobs
  `test` (one per platform) and `docs`. Task 3 adds `inventory`.

- [ ] **Step 1: Write `downstream`** without a build script: package
  `downstream`, `publish = false`, depending on `rdkit = { path = ".." }`. Its
  `main` prints `RDKit {rdkit_version()}`.

- [ ] **Step 2: Watch it fail to load.**
  Run: `cargo build -p downstream && env -u LD_LIBRARY_PATH -u
  DYLD_LIBRARY_PATH -u DYLD_FALLBACK_LIBRARY_PATH ./target/debug/downstream`
  Expected: FAIL, `Library not loaded: @rpath/libRDKit…` on macOS or `error
  while loading shared libraries` on Linux. This proves the next step is what
  makes it run.

- [ ] **Step 3: Add `downstream/build.rs`**, whose `main` is the one line the
  README documents for users:
  `println!("cargo::rustc-link-arg=-Wl,-rpath,{}",
  std::env::var("DEP_RDKIT_LIBDIR").unwrap());`

- [ ] **Step 4: Run it again.**
  Expected: prints a line starting `RDKit 2026.09.`.

- [ ] **Step 5: Watch docs fail without RDKit.**
  Run: `env -u CONDA_PREFIX -u RDKIT_PREFIX DOCS_RS=1 cargo doc --no-deps`
  Expected: FAIL with the missing-RDKit panic.

- [ ] **Step 6: Make `build.rs` return first thing when `DOCS_RS` is set**, so
  docs build without compiling or linking C++. Run Step 5 again.
  Expected: PASS.

- [ ] **Step 7: Write `.github/workflows/ci.yml`.** Triggers: push and
  pull_request. No caching of conda packages, Cargo, or build outputs.
  - `test`: matrix `ubuntu-24.04`, `ubuntu-24.04-arm`, `macos-15`.
    `mamba-org/setup-micromamba@v2` with `environment.yml`;
    `dtolnay/rust-toolchain@stable` with `rustfmt` and `clippy`; shell
    `bash -el {0}`. It runs `cargo fmt --check`, `cargo clippy --workspace
    --all-targets -- -D warnings`, `cargo test --workspace`, `cmake -P
    tests/cmake/version_check.cmake`, Step 4's command checked with `grep "^RDKit
    2026.09."`, and Task 1 Step 13's command, which must fail the build and
    print the message.
  - `docs`: `ubuntu-24.04` with no conda environment, running Step 5's command.

- [ ] **Step 8: Document use** in `README.md`: the packages to install, the
  Step 3 line, and why it is needed (conda does not set `LD_LIBRARY_PATH`, and
  a dependency's build script cannot put an rpath in another crate's binary).

- [ ] **Step 9: Commit**: `git commit -m "Add CI, a downstream crate, and a
  docs.rs build"`.

- [ ] **Step 10: Publish the repository, after asking.** Ask the user whether
  to create `rdkit-rust` on GitHub and whether it is public or private. Arm64
  Linux runners are free only for public repositories. With a yes, create it,
  push `main` and `core-slice`, open a draft PR from `core-slice`, and read the
  workflow's status once. Do not wait on it.

### Task 3: Inventory, manifest, and parity check

**Files:**

- Create: `inventory/inventory.py`, `inventory/check.py`,
  `inventory/manifest.toml`, `inventory/inventory-2026.09.json` (generated),
  `inventory/test_check.py`, `inventory/test_inventory_contents.py`
- Create: `tests/manifest_paths.rs` (generated)
- Modify: `.github/workflows/ci.yml` (job `inventory`), `CONVENTIONS.md`,
  `README.md` (regenerating the inventory)

**Interfaces:**

- Produces:
  - Inventory JSON: `{"rdkit_version": "2026.09.1", "entries": [{"python":
    "rdkit.Chem.rdchem.Mol.GetAtomWithIdx", "kind": "method", "overloads":
    ["<nanobind signature>", ...]}]}`, sorted by `python`. Kinds:
    `module_function`, `module_attribute`, `class`, `method`, `static_method`,
    `property`, `enum_value`, `flag_value`. `overloads` holds nanobind's
    per-overload signatures (`__nb_signature__`) and is empty for
    non-callables.
  - Manifest TOML: `[modules] complete = [<module names>]`, then `[[entry]]`
    tables with `python` and exactly one of `rust` (a list of paths) or
    `reason`, plus optional `rust_kind = "field"`, `name_exception`, and
    `overloads_note`.
  - `check.py`: `snake_case(name: str) -> str`,
    `upper_camel_case(name: str) -> str`,
    `check(inventory: dict, manifest: dict) -> list[str]` (problems), and
    `paths_source(manifest: dict) -> str`. CLI: `python3 inventory/check.py
    [--inventory PATH] [--manifest PATH] [--write-paths PATH]`. It prints
    `mapped <m> of <n> entries (<r> with a reason); complete modules: <c>` and
    exits 1 if there are problems.

- [ ] **Step 1: Write the failing tests** in `inventory/test_check.py`. Cases:
  - `snake_case`: `GetAtomWithIdx` → `get_atom_with_idx`, `allowCXSMILES` →
    `allow_cxsmiles`, `MolToSmiles` → `mol_to_smiles`, `rdkitVersion` →
    `rdkit_version`, `Is3D` → `is3_d` (the case `name_exception` exists for).
  - `upper_camel_case`: `SINGLE` → `Single`, `CHI_TETRAHEDRAL_CW` →
    `ChiTetrahedralCw`, `SP3D2` → `Sp3d2`.
  - `check` returns exactly one problem for each of: a manifest name missing
    from the inventory; an entry with both `rust` and `reason`; an unmapped
    entry in a `complete` module; a Rust name that is neither the expected name
    nor that name followed by `_`; a method with two overloads and one Rust
    path.
  - `check` returns no problems for: an unmapped entry in an incomplete
    module; `get_conformer_with_id` for `GetConformer`; a `name_exception`; an
    `overloads_note`; `__init__` mapped to `new`, `from_num`, `default`, or
    `clone`; `BondType.SINGLE` mapped to `BondType::Single`;
    `SanitizeFlags.SANITIZE_ALL` mapped to `SanitizeFlags::SANITIZE_ALL`.
  - `paths_source` emits `let _ = <path>;` for functions, methods, enum values,
    and constants; `let _: Option<<path>> = None;` for classes; and
    `let _ = |value: &<type>| &value.<field>;` for `rust_kind = "field"`. The
    statements sit inside one `#[test] fn manifest_paths_exist()`.

  Run: `python3 -m pytest inventory/test_check.py`
  Expected: FAIL, `check` not found.

- [ ] **Step 2: Implement `check.py`.** `snake_case` is:

  ```python
  def snake_case(name: str) -> str:
      name = re.sub(r"([a-z0-9])([A-Z])", r"\1_\2", name)
      name = re.sub(r"([A-Z]+)([A-Z][a-z])", r"\1_\2", name)
      return name.lower()
  ```

  `upper_camel_case` capitalizes each `_`-separated part of
  `snake_case(name)` and joins them. The expected last path segment is
  `snake_case` for functions, methods, properties, and module attributes, and
  for `rust_kind = "field"`; the class name for classes;
  `upper_camel_case` for enum values; and the Python name for flag values.
  `__init__` may map to `new`, `default`, `clone`, or `from_*`. Other dunder
  names skip the naming check. Parse TOML with `tomllib`.

- [ ] **Step 3: Run the check tests.**
  Expected: PASS.

- [ ] **Step 4: Write `inventory.py`.** Arguments: `--rdkit-path DIR`
  (prepended to `sys.path`) and `--output PATH`. Walk `rdkit` with
  `pkgutil.walk_packages`, keep modules whose `__file__` ends in one of
  `importlib.machinery.EXTENSION_SUFFIXES`, and record the objects whose
  `__module__` is that module. Members of `enum.Flag` subclasses are
  `flag_value`; members of other `enum.Enum` subclasses are `enum_value`.

- [ ] **Step 5: Generate the inventory.** Build RDKit at `Release_2026_09_1`
  in its own worktree (`git -C ~/rdkit/rdkit worktree add
  ~/rdkit/rdkit-release-2026.09.1 Release_2026_09_1`), in an environment with
  RDKit's build dependencies and nanobind, configured with
  `-DRDK_BUILD_NANOBIND_WRAPPERS=ON -DRDK_BUILD_BOOST_PYTHON_WRAPPERS=OFF
  -DRDK_INSTALL_INTREE=ON -DRDK_BUILD_CPP_TESTS=OFF` plus conda-forge's optional
  modules (Avalon, Cairo, Contrib, FreeSASA, InChI, xyz2mol, YAeHMOP). The
  build is long: run it in the background. Then run `python3
  inventory/inventory.py --rdkit-path ~/rdkit/rdkit-release-2026.09.1
  --output inventory/inventory-2026.09.json`. Put these commands in the README.

- [ ] **Step 6: Write `inventory/test_inventory_contents.py`**, with fixtures
  `inventory` (the checked-in JSON) and `by_name` (its entries keyed by
  `python`):

  ```python
  def test_core_entries(by_name, inventory):
      assert inventory["rdkit_version"] == "2026.09.1"
      assert by_name["rdkit.rdBase.rdkitVersion"]["kind"] == "module_attribute"
      assert by_name["rdkit.rdBase.DisableLog"]["kind"] == "module_function"
      assert by_name["rdkit.Chem.rdchem.Mol"]["kind"] == "class"
      assert by_name["rdkit.Chem.rdchem.Mol.GetAtomWithIdx"]["kind"] == "method"
      assert len(by_name["rdkit.Chem.rdmolfiles.MolFromSmiles"]["overloads"]) == 2
      assert by_name["rdkit.Chem.rdchem.BondType.SINGLE"]["kind"] == "enum_value"
      assert by_name["rdkit.Chem.rdmolfiles.SmilesParserParams.sanitize"]["kind"] == "property"
  ```

  Add `SanitizeFlags.SANITIZE_ALL` with kind `flag_value` if the nanobind
  build defines `SanitizeFlags` as an `enum.Flag`. If it does not, record that
  in `CONVENTIONS.md` under N4, and give its values `name_exception` in Task 10.
  Run: `python3 -m pytest inventory`
  Expected: PASS.

- [ ] **Step 7: Write the manifest** with `complete = []` and entries for the
  four `rdBase` names from Task 1. Run `python3 inventory/check.py
  --write-paths tests/manifest_paths.rs`, then `cargo test --test
  manifest_paths`.
  Expected: `mapped 4 of <n> entries`; the test passes.

- [ ] **Step 8: Add CI job `inventory`** (`ubuntu-24.04`, Python 3.12,
  pytest): `python3 -m pytest inventory`, then the Step 7 command, then
  `git diff --exit-code tests/manifest_paths.rs`.

- [ ] **Step 9: Add rules to `CONVENTIONS.md`:**
  - **I1.** A commit that binds a Python name adds its manifest entry and
    regenerates `tests/manifest_paths.rs`.
  - **I2.** `reason` marks a name with no Rust counterpart by design (Python
    protocol methods, pickling hooks, APIs RDKit has deprecated). A name that is
    not bound yet has no entry.
  - **I3.** A module joins `complete` once every inventory entry in it is
    mapped; CI then holds it there.

- [ ] **Step 10: Commit**: `git commit -m "Measure parity against the
  nanobind wrappers"`.

### Task 4: Errors, `Mol`, and SMILES

**Files:**

- Create: `wrapper/include/rdkit-rust/{errors,mol,molfiles}.h`,
  `wrapper/src/{errors,mol,molfiles}.cpp`, `src/ffi/{errors,mol,molfiles}.rs`,
  `src/chem/mod.rs`, `src/chem/rdchem/mod.rs`, `src/chem/rdchem/mol.rs`,
  `src/chem/rdmolfiles.rs`
- Modify: `src/error.rs`, `src/lib.rs`, `src/ffi/rd_base.rs` (include
  `errors.h` first), `build.rs`, `wrapper/CMakeLists.txt` (targets `GraphMol`,
  `SmilesParse`), `CONVENTIONS.md`, manifest
- Test: `tests/errors.rs`, `tests/smiles.rs`

**Interfaces:**

- Produces:

  ```rust
  pub type Result<T> = std::result::Result<T, Error>;   // rdkit::Result

  #[derive(Clone, Debug, PartialEq)]
  #[non_exhaustive]
  pub enum Error {                                       // rdkit::Error
      Parse { message: String },
      AtomValence { atom_idx: u32, message: String },
      AtomKekulize { atom_idx: u32, message: String },
      Kekulize { atom_indices: Vec<u32>, message: String },
      MolSanitize { message: String },
      Index { index: u32, len: u32 },
      Key { key: String },
      Value { message: String },
      Runtime { message: String },
      Os { message: String },
  }
  ```

  `Display` (including the message), `std::error::Error`, and
  `From<cxx::Exception>`. `rdkit::error` becomes public, with `Error` and
  `Result` re-exported at the crate root. Hidden test hooks in `rdkit::error`:
  `__raise_for_tests(kind: &str) -> Result<()>` and
  `__raise_unexpected_for_tests()`.
- `rdkit::chem::rdchem::Mol`: `Default`, `get_num_atoms(&self) -> u32`,
  `get_num_bonds(&self) -> u32`, `get_num_heavy_atoms(&self) -> u32`, `Clone`
  (C++ copy), `Debug` (`Mol { num_atoms: 3, num_bonds: 2 }`), `Send`, not
  `Sync`. Crate-internal: `from_ffi(UniquePtr<ffi::RWMol>) -> Mol`,
  `as_ffi(&self) -> &ffi::RWMol`, `pin_mut(&mut self) -> Pin<&mut ffi::RWMol>`.
- `rdkit::chem::rdmolfiles::{mol_from_smiles(smiles: &str) -> Result<Mol>,
  mol_to_smiles(mol: &Mol) -> Result<String>}`.
- C++: `rdkit_rust::IndexError(unsigned index, unsigned len)`, deriving from
  `std::exception`, and a `rust::behavior::trycatch` in `errors.h` that
  encodes each caught exception as `\x1Frdkit\x1F<Variant>\x1F<payload>\x1F<what>`.
  The payload is the atom index (AtomValence, AtomKekulize), space-separated
  indices (Kekulize), `<index> <len>` (Index), the key (Key), and empty
  otherwise. A message without the prefix becomes `Runtime` whole.

- [ ] **Step 1: Write the failing error tests** in `tests/errors.rs`. The C++
  helper behind `__raise_for_tests` throws one exception per kind, with message
  `test <kind>`, and payloads atom 3, atom 2, atoms `[0, 1, 2, 3, 4]`, index 9
  of 3, and key `nope`.

  ```rust
  use rdkit::Error;
  use rdkit::error::{__raise_for_tests as raise, __raise_unexpected_for_tests};

  fn message(kind: &str) -> String { format!("test {kind}") }

  #[test]
  fn each_rdkit_exception_maps_to_its_variant() {
      assert_eq!(raise("atom_valence").unwrap_err(),
                 Error::AtomValence { atom_idx: 3, message: message("atom_valence") });
      assert_eq!(raise("atom_kekulize").unwrap_err(),
                 Error::AtomKekulize { atom_idx: 2, message: message("atom_kekulize") });
      assert_eq!(raise("kekulize").unwrap_err(),
                 Error::Kekulize { atom_indices: vec![0, 1, 2, 3, 4], message: message("kekulize") });
      assert_eq!(raise("mol_sanitize").unwrap_err(), Error::MolSanitize { message: message("mol_sanitize") });
      assert_eq!(raise("index").unwrap_err(), Error::Index { index: 9, len: 3 });
      assert_eq!(raise("key").unwrap_err(), Error::Key { key: "nope".into() });
      assert_eq!(raise("value").unwrap_err(), Error::Value { message: message("value") });
      assert_eq!(raise("conformer").unwrap_err(), Error::Value { message: message("conformer") });
      assert_eq!(raise("bad_file").unwrap_err(), Error::Os { message: message("bad_file") });
      assert_eq!(raise("smiles_parse").unwrap_err(), Error::Parse { message: message("smiles_parse") });
      assert_eq!(raise("std").unwrap_err(), Error::Runtime { message: message("std") });
      assert!(matches!(raise("invariant").unwrap_err(),
                       Error::Runtime { message } if message.contains("test invariant")));
  }

  #[test]
  fn display_includes_the_message() {
      assert!(raise("value").unwrap_err().to_string().contains("test value"));
  }

  #[test]
  #[should_panic(expected = "unexpected RDKit exception: test std")]
  fn unexpected_exceptions_panic() { __raise_unexpected_for_tests(); }
  ```

  Run: `cargo test --test errors`
  Expected: FAIL to compile.

- [ ] **Step 2: Implement `errors.h`, `errors.cpp`, `src/ffi/errors.rs`, and
  `src/error.rs`.** `trycatch` catches the most derived type first:

  | C++ exception | Variant |
  | --- | --- |
  | `RDKit::AtomValenceException` | `AtomValence` |
  | `RDKit::AtomKekulizeException` | `AtomKekulize` |
  | `RDKit::KekulizeException` | `Kekulize` |
  | `RDKit::MolSanitizeException` | `MolSanitize` |
  | `rdkit_rust::IndexError` | `Index` |
  | `RDKit::KeyErrorException` | `Key` |
  | `RDKit::ValueErrorException`, `RDKit::ConformerException` | `Value` |
  | `RDKit::BadFileException` | `Os` |
  | `RDKit::SmilesParseException`, `RDKit::FileParseException` | `Parse` |
  | `Invar::Invariant`, any other `std::exception` | `Runtime` |

  `__raise_unexpected_for_tests` passes the `std` kind through `expect_ok`.

- [ ] **Step 3: Run the error tests.**
  Expected: PASS.

- [ ] **Step 4: Write the failing SMILES tests** in `tests/smiles.rs`:

  ```rust
  use rdkit::Error;
  use rdkit::chem::{Mol, mol_from_smiles, mol_to_smiles};

  #[test]
  fn round_trip_canonicalizes() {
      let mol = mol_from_smiles("OCC").unwrap();
      assert_eq!((mol.get_num_atoms(), mol.get_num_bonds()), (3, 2));
      assert_eq!(mol_to_smiles(&mol).unwrap(), "CCO");
  }

  #[test]
  fn empty_smiles_is_an_empty_molecule() {
      let mol = mol_from_smiles("").unwrap();
      assert_eq!(mol.get_num_atoms(), 0);
      assert_eq!(mol_to_smiles(&mol).unwrap(), "");
  }

  #[test]
  fn unparsable_smiles_is_a_parse_error() {
      assert!(matches!(mol_from_smiles("C1CC"),
                       Err(Error::Parse { message }) if message.contains("C1CC")));
  }

  #[test]
  fn sanitization_errors_keep_their_atoms() {
      assert!(matches!(mol_from_smiles("C(C)(C)(C)(C)C"),
                       Err(Error::AtomValence { atom_idx: 0, message }) if message.contains("valence")));
      assert!(matches!(mol_from_smiles("c1cccc1"),
                       Err(Error::Kekulize { atom_indices, .. }) if atom_indices == [0, 1, 2, 3, 4]));
      assert!(matches!(mol_from_smiles("cC"), Err(Error::AtomKekulize { atom_idx: 0, .. })));
  }

  #[test]
  fn clone_outlives_the_original() {
      let mol = mol_from_smiles("OCC").unwrap();
      let copy = mol.clone();
      drop(mol);
      assert_eq!(mol_to_smiles(&copy).unwrap(), "CCO");
  }

  #[test]
  fn debug_shows_counts() {
      assert_eq!(format!("{:?}", mol_from_smiles("OCC").unwrap()), "Mol { num_atoms: 3, num_bonds: 2 }");
  }

  #[test]
  fn mol_is_send() {
      fn assert_send<T: Send>() {}
      assert_send::<Mol>();
  }
  ```

  Add a `compile_fail` doctest to `Mol` asserting it is not `Sync`.

- [ ] **Step 5: Implement `Mol`, `mol_from_smiles`, and `mol_to_smiles`.**
  The SMILES wrapper calls `RDKit::SmilesToMol`, which throws the sanitization
  exceptions itself. When it returns `nullptr` (a parse failure), the wrapper
  throws `SmilesParseException` with a message containing the input. `Mol`
  holds a `UniquePtr<ffi::RWMol>` that is never null. `unsafe impl Send` for
  `ffi::RWMol` carries a comment citing S1.

- [ ] **Step 6: Run the tests.**
  Run: `cargo test --test smiles --test errors && cargo test --doc`
  Expected: PASS.

- [ ] **Step 7: Add rules to `CONVENTIONS.md`:**
  - **E1.** A public function returns `Result` when the nanobind call can
    raise for some input reachable through the safe Rust API: an index, a key,
    unparsable text, a failed sanitization, a missing property cache or ring
    info, a missing conformer, invalid UTF-8. Otherwise it returns the value,
    and an unexpected C++ exception panics through `expect_ok`.
  - **E2.** Parsers that return `None` in Python return `Err(Error::Parse)`. A
    parse that fails sanitization returns the sanitization variant.
  - **E3.** Index arguments are checked in the C++ wrapper, which throws
    `rdkit_rust::IndexError`. (Python raises `RuntimeError` from RDKit's own
    range check there.)
  - **E4.** The exception table above.
  - **E5.** Every bridge's `include!` list starts with `"rdkit-rust/errors.h"`.
  - **W1.** Every Rust molecule owns a C++ `RDKit::RWMol`, so in-place
    operations can take `RWMol&` without a cast.
  - **S1.** `Mol` and `RWMol` are `Send` and not `Sync`. RDKit writes computed
    properties through `const` molecules (`MolToSmiles` sets
    `_smilesAtomOutputOrder` in `SmilesWrite.cpp`), so two threads reading one
    molecule race. Parallel code gives each thread its own molecule.
  - **D1.** Doc examples are `no_run` or `compile_fail`; behavior is tested
    under `tests/`.
  - **A1.** Required Python parameters become positional Rust parameters, in
    Python's order.
  - **A2.** A Python signature with optional parameters is bound as `name`
    (using Python's defaults) plus, for exactly one optional parameter,
    `name_with_<param>`, or for two or more, `name_with_params(&<Name>Params)`.
    `<Name>Params` is named for the Python function or method, has the optional
    parameters as snake_case fields, and implements `Default` with Python's
    defaults, pinned by a test.
  - **A3.** When Python also accepts a params object (an overload taking, say,
    `SmilesParserParams`), `name_with_params` takes the Rust mirror of that
    class instead. Keyword arguments the class lacks become extra fields on
    it. The mirror's `Default` reads a default-constructed C++ object, so it
    cannot drift.
  - **A4.** An optional parameter that changes the return type gets its own
    function, `name_<param>` (`GetMolFrags(asMols=True)` →
    `get_mol_frags_as_mols`), and does not count toward A2. Python output
    arguments (lists the call fills in) are not bound; the return value carries
    that information.
  - **A5.** Overloads that differ by an argument's type: the first in
    nanobind's order keeps the name, and the others append `_<type>` in
    snake_case. Constructors: a zero-argument `__init__` is `Default`; a copy
    `__init__` is `Clone`; the first other `__init__` is `new`; the rest are
    `from_<first parameter>`.
  - **A6.** `_mut` comes last: `get_conformer_with_id_mut`.
  - **T1.** Atom and bond indices are `u32`; conformer ids are `i32`, since
    they can be −1. A Python `int` otherwise follows the C++ parameter type.
    `float` → `f64`. `str` → `&str` in, `String` out. File names →
    `impl AsRef<Path>`. Tuples of indices → `Vec<u32>`, nested →
    `Vec<Vec<u32>>`. `None`-able results → `Option`. NumPy coordinate arrays →
    `Vec<Point3D>`. `dict` → `BTreeMap<String, _>`.

- [ ] **Step 8: Update the manifest.** Add `Mol`, its constructor and
  `__copy__`/`__deepcopy__` (→ `clone`), the three counts, `MolFromSmiles`
  (with `overloads_note = "params overload in Task 7"`), and `MolToSmiles`,
  plus any `_with_` functions A2 requires for these signatures. Regenerate
  `tests/manifest_paths.rs`.

- [ ] **Step 9: Commit**: `git commit -m "Map RDKit exceptions to Error and
  parse SMILES"`.

### Task 5: Atom and bond views

**Files:**

- Create: `wrapper/include/rdkit-rust/{atom,bond}.h`,
  `wrapper/src/{atom,bond}.cpp`, `src/ffi/{atom,bond}.rs`,
  `src/chem/rdchem/{atom,bond,enums}.rs`
- Modify: `src/chem/rdchem/mol.rs`, `build.rs`, `wrapper/CMakeLists.txt`,
  `CONVENTIONS.md`, manifest
- Test: `tests/views.rs`

**Interfaces:**

- Consumes: `Mol::as_ffi`, `Error::Index`.
- Produces:
  - Enums `BondType`, `HybridizationType`, and `ChiralType`, with every value
    of the Python enums in UpperCamelCase, deriving `Clone`, `Copy`, `Debug`,
    `PartialEq`, `Eq`, and `Hash`. Views implement `Debug` (their index), so
    `Result<AtomRef>` supports `unwrap_err`. The bridge binds the C++ enums so cxx
    checks their values. If cxx rejects a nested C++ enum through a `using`
    alias in `rdkit_rust`, cross as `i32` instead, and add a test that sends
    every variant through C++ and back.
  - `AtomRef<'m>` (`Copy`, holding `&'m Mol` and `&'m ffi::Atom`):
    `get_idx() -> u32`, `get_atomic_num() -> i32`, `get_symbol() -> String`,
    `get_formal_charge() -> i32`, `get_is_aromatic() -> bool`,
    `get_isotope() -> u32`, `get_degree() -> u32`,
    `get_num_explicit_hs() -> u32`, `get_hybridization() -> HybridizationType`,
    `get_chiral_tag() -> ChiralType`, `is_in_ring() -> bool` (reproducing the
    nanobind wrapper's ring-info handling), `get_total_degree() -> Result<u32>`,
    `get_total_num_hs() -> Result<u32>`, `get_num_implicit_hs() -> Result<u32>`,
    `get_total_valence() -> Result<u32>`,
    `get_neighbors() -> Vec<AtomRef<'m>>`, `get_bonds() -> Vec<BondRef<'m>>`,
    and `get_owning_mol() -> &'m Mol`.
  - `BondRef<'m>` (`Copy`): `get_idx() -> u32`, `get_bond_type() -> BondType`,
    `get_bond_type_as_double() -> f64`, `get_begin_atom_idx() -> u32`,
    `get_end_atom_idx() -> u32`, `get_begin_atom() -> AtomRef<'m>`,
    `get_end_atom() -> AtomRef<'m>`, `get_other_atom_idx(this_idx: u32) ->
    Result<u32>`, `get_is_aromatic() -> bool`, `get_is_conjugated() -> bool`,
    and `get_owning_mol() -> &'m Mol`.
  - On `Mol`: `get_atom_with_idx(&self, idx: u32) -> Result<AtomRef<'_>>`,
    `get_bond_with_idx(&self, idx: u32) -> Result<BondRef<'_>>`,
    `get_bond_between_atoms(&self, idx1: u32, idx2: u32) ->
    Result<Option<BondRef<'_>>>`, `get_atoms(&self) -> Atoms<'_>`, and
    `get_bonds(&self) -> Bonds<'_>`. The iterators implement `Iterator`,
    `ExactSizeIterator`, and `DoubleEndedIterator`.
  - Macros `atom_getters!` and `bond_getters!`, which stamp the
    owner-independent getters onto a type that provides
    `fn raw(&self) -> &ffi::Atom` (or `&ffi::Bond`). Task 6 reuses them.

- [ ] **Step 1: Write the failing tests** in `tests/views.rs`:

  ```rust
  use rdkit::Error;
  use rdkit::chem::{BondType, ChiralType, HybridizationType, mol_from_smiles};

  #[test]
  fn atom_getters() {
      let mol = mol_from_smiles("OCC").unwrap();
      let symbols: Vec<_> = mol.get_atoms().map(|a| a.get_symbol()).collect();
      assert_eq!(symbols, ["O", "C", "C"]);
      let degrees: Vec<_> = mol.get_atoms().map(|a| a.get_degree()).collect();
      assert_eq!(degrees, [1, 2, 1]);
      let hs: Vec<_> = mol.get_atoms().map(|a| a.get_total_num_hs().unwrap()).collect();
      assert_eq!(hs, [1, 2, 3]);
      let atom = mol.get_atom_with_idx(1).unwrap();
      let neighbors: Vec<_> = atom.get_neighbors().iter().map(|a| a.get_idx()).collect();
      assert_eq!(neighbors, [0, 2]);
      assert!(std::ptr::eq(atom.get_owning_mol(), &mol));
  }

  #[test]
  fn enums() {
      let mol = mol_from_smiles("C=CO").unwrap();
      let types: Vec<_> = mol.get_bonds().map(|b| b.get_bond_type()).collect();
      assert_eq!(types, [BondType::Double, BondType::Single]);
      let hybridizations: Vec<_> = mol.get_atoms().map(|a| a.get_hybridization()).collect();
      assert_eq!(hybridizations, [HybridizationType::Sp2; 3]);
      let chiral = mol_from_smiles("F[C@H](Cl)Br").unwrap();
      assert_eq!(chiral.get_atom_with_idx(1).unwrap().get_chiral_tag(), ChiralType::ChiTetrahedralCcw);
  }

  #[test]
  fn bonds_and_lookups() {
      let mol = mol_from_smiles("C=CO").unwrap();
      let ends: Vec<_> = mol.get_bonds().map(|b| (b.get_begin_atom_idx(), b.get_end_atom_idx())).collect();
      assert_eq!(ends, [(0, 1), (1, 2)]);
      assert_eq!(mol.get_bond_between_atoms(0, 1).unwrap().unwrap().get_idx(), 0);
      assert!(mol.get_bond_between_atoms(0, 2).unwrap().is_none());
      assert_eq!(mol.get_bond_between_atoms(0, 99).unwrap_err(), Error::Index { index: 99, len: 3 });
      assert_eq!(mol.get_atom_with_idx(99).unwrap_err(), Error::Index { index: 99, len: 3 });
      assert_eq!(mol.get_bond_with_idx(99).unwrap_err(), Error::Index { index: 99, len: 2 });
  }

  #[test]
  fn iterators() {
      let mol = mol_from_smiles("OCC").unwrap();
      assert_eq!(mol.get_atoms().len(), 3);
      assert_eq!(mol.get_atoms().rev().map(|a| a.get_idx()).collect::<Vec<_>>(), [2, 1, 0]);
      let empty = mol_from_smiles("").unwrap();
      assert_eq!((empty.get_atoms().count(), empty.get_bonds().count()), (0, 0));
  }
  ```

  Add a `compile_fail` doctest showing that an `AtomRef` cannot outlive its
  `Mol`.
  Run: `cargo test --test views`
  Expected: FAIL to compile.

- [ ] **Step 2: Implement the C++ wrappers, bridges, enums, views, and
  iterators.**

- [ ] **Step 3: Run the tests.**
  Run: `cargo test --test views && cargo test --doc`
  Expected: PASS.

- [ ] **Step 4: Add rules to `CONVENTIONS.md`:**
  - **W2.** Shared views are `XRef<'m>` (`Copy`, holding `&'m Mol` and
    `&'m ffi::X`). Exclusive views are `XMut<'m>` (holding `&'m mut Mol` and
    an index). An owned `X` exists only when Python has a constructor.
  - **W3.** Getters are written once, in a `macro_rules!` that stamps them on
    every type that has them (owned, `Ref`, `Mut`), all calling wrappers that
    take `&ffi::X`. Setters work the same way, on owned and `Mut` types.
  - **W4.** Methods that need the owning molecule (neighbors, bonds,
    `get_owning_mol`) exist only on views.
  - **W5.** Python sequences become iterators named for the result (`Atoms`,
    `Bonds`) that implement `ExactSizeIterator` and `DoubleEndedIterator`.
    There are no mutable iterators. Code mutates by index, because two live
    `XMut`s into one molecule would alias.

- [ ] **Step 5: Update the manifest** for every Python name bound here,
  regenerate the paths, and commit: `git commit -m "Add atom and bond views"`.

### Task 6: Mutation and editing

**Files:**

- Create: `src/chem/rdchem/rwmol.rs`
- Modify: the atom, bond, and mol wrappers and bridges;
  `src/chem/rdchem/{atom,bond,mol}.rs`; `CONVENTIONS.md`; manifest
- Test: `tests/editing.rs`

**Interfaces:**

- Consumes: `atom_getters!`, `bond_getters!`, `Mol::pin_mut`.
- Produces:
  - `Atom` (owned): `new(what: &str) -> Result<Atom>`,
    `from_num(num: u32) -> Atom`, `Clone`, and the owner-independent getters
    and the setters.
  - `AtomMut<'m>`: every `AtomRef` getter plus `set_atomic_num(i32)`,
    `set_formal_charge(i32)`, `set_is_aromatic(bool)`, `set_isotope(u32)`,
    `set_num_explicit_hs(u32)`, `set_no_implicit(bool)`,
    `set_hybridization(HybridizationType)`, `set_chiral_tag(ChiralType)`,
    `update_property_cache() -> Result<()>`, and
    `update_property_cache_with_strict(strict: bool) -> Result<()>`.
  - `BondMut<'m>`: every `BondRef` getter plus `set_bond_type(BondType)`,
    `set_is_aromatic(bool)`, and `set_is_conjugated(bool)`.
  - On `Mol`: `get_atom_with_idx_mut(&mut self, idx: u32) ->
    Result<AtomMut<'_>>`, `get_bond_with_idx_mut(&mut self, idx: u32) ->
    Result<BondMut<'_>>`, `update_property_cache(&mut self) -> Result<()>`, and
    `update_property_cache_with_strict(&mut self, strict: bool) -> Result<()>`.
  - `RWMol` (wrapping a `Mol`): `Default`, `Deref<Target = Mol>`, `DerefMut`,
    `From<Mol>`, and `From<RWMol> for Mol` (both moves, no copy),
    `get_mol(&self) -> Mol` (a copy), `add_atom(&mut self, atom: &Atom) ->
    u32`, `add_bond(&mut self, begin_atom_idx: u32, end_atom_idx: u32) ->
    Result<u32>`, `add_bond_with_order(&mut self, begin_atom_idx: u32,
    end_atom_idx: u32, order: BondType) -> Result<u32>` (both return the new
    bond count, as Python does), `remove_atom(&mut self, idx: u32) ->
    Result<()>`, `remove_bond(&mut self, idx1: u32, idx2: u32) -> Result<()>`,
    `replace_atom(&mut self, index: u32, new_atom: &Atom) -> Result<()>`, and
    `replace_atom_with_params(&mut self, index: u32, new_atom: &Atom, params:
    &ReplaceAtomParams) -> Result<()>`, where
    `ReplaceAtomParams { update_label: false, preserve_props: false }`.
    Parameter names follow the inventory where it differs.

- [ ] **Step 1: Write the failing tests** in `tests/editing.rs`:

  ```rust
  use rdkit::Error;
  use rdkit::chem::{Atom, BondType, Mol, RWMol, mol_from_smiles, mol_to_smiles};

  fn smiles(mol: &Mol) -> String { mol_to_smiles(mol).unwrap() }

  #[test]
  fn build_atom_by_atom() {
      let mut rw = RWMol::default();
      assert_eq!(rw.add_atom(&Atom::from_num(6)), 0);
      assert_eq!(rw.add_atom(&Atom::new("O").unwrap()), 1);
      assert_eq!(rw.add_bond_with_order(0, 1, BondType::Single).unwrap(), 1);
      assert_eq!(smiles(&rw), "CO");
      assert_eq!(rw.add_bond(0, 5).unwrap_err(), Error::Index { index: 5, len: 2 });
  }

  #[test]
  fn property_cache_is_required_and_reported() {
      let mut rw = RWMol::default();
      rw.add_atom(&Atom::from_num(6));
      rw.add_atom(&Atom::from_num(8));
      rw.add_bond_with_order(0, 1, BondType::Single).unwrap();
      assert!(matches!(rw.get_atom_with_idx(0).unwrap().get_total_num_hs(),
                       Err(Error::Runtime { message }) if message.contains("calcImplicitValence")));
      rw.update_property_cache().unwrap();
      assert_eq!(rw.get_atom_with_idx(0).unwrap().get_total_num_hs().unwrap(), 3);
  }

  #[test]
  fn edit_through_views() {
      let mut acid = mol_from_smiles("CC(=O)O").unwrap();
      acid.get_atom_with_idx_mut(3).unwrap().set_formal_charge(-1);
      acid.update_property_cache().unwrap();
      assert_eq!(smiles(&acid), "CC(=O)[O-]");

      let mut ethane = mol_from_smiles("CC").unwrap();
      for i in 0..ethane.get_num_atoms() {
          ethane.get_atom_with_idx_mut(i).unwrap().set_isotope(13);
      }
      assert_eq!(smiles(&ethane), "[13CH3][13CH3]");

      let mut ethene = mol_from_smiles("CC").unwrap();
      ethene.get_bond_with_idx_mut(0).unwrap().set_bond_type(BondType::Double);
      ethene.update_property_cache().unwrap();
      assert_eq!(smiles(&ethene), "C=C");
  }

  #[test]
  fn rwmol_edits() {
      let mut rw = RWMol::from(mol_from_smiles("OCC").unwrap());
      rw.remove_atom(0).unwrap();
      assert_eq!(smiles(&rw), "CC");
      rw.replace_atom(0, &Atom::from_num(7)).unwrap();
      assert_eq!(smiles(&rw), "CN");
      let copy = rw.get_mol();
      rw.remove_bond(0, 1).unwrap();
      assert_eq!((smiles(&rw), smiles(&copy)), ("C.N".to_string(), "CN".to_string()));
  }

  #[test]
  fn atom_constructors() {
      assert_eq!(Atom::new("C").unwrap().get_atomic_num(), 6);
      assert!(matches!(Atom::new("Xx"), Err(Error::Runtime { message }) if message.contains("Xx")));
  }

  #[test]
  fn rwmol_derefs_to_mol_and_is_send() {
      fn count(mol: &Mol) -> u32 { mol.get_num_atoms() }
      assert_eq!(count(&RWMol::from(mol_from_smiles("CC").unwrap())), 2);
      fn assert_send<T: Send>() {}
      assert_send::<RWMol>();
  }
  ```

  Run: `cargo test --test editing`
  Expected: FAIL to compile.

- [ ] **Step 2: Implement it.** `AtomMut` and `BondMut` hold `&'m mut Mol`
  and an index, and fetch the C++ object per call (W2), so they need no
  `unsafe`. `RWMol` is `struct RWMol { mol: Mol }`.

- [ ] **Step 3: Run the tests.**
  Expected: PASS.

- [ ] **Step 4: Update the manifest**, regenerate the paths, and commit:
  `git commit -m "Add mutable views and RWMol editing"`.

### Task 7: SMILES, SMARTS, and molfiles with parameters

**Files:**

- Modify: `wrapper/include/rdkit-rust/molfiles.h`,
  `wrapper/src/molfiles.cpp`, `src/ffi/molfiles.rs`,
  `src/chem/rdmolfiles.rs`, `wrapper/CMakeLists.txt` (target `FileParsers`),
  `CONVENTIONS.md`, manifest
- Test: `tests/molfiles.rs`

**Interfaces:**

- Consumes: `Mol`, `Error`.
- Produces in `rdkit::chem::rdmolfiles`, with fields and defaults following
  the inventory (the values below are from RDKit 2026.03; the inventory wins
  where they differ):
  - `SmilesParserParams { allow_cxsmiles: bool, debug_parse: i32,
    parse_name: bool, remove_hs: bool, sanitize: bool, strict_cxsmiles: bool,
    replacements: BTreeMap<String, String> }` (A3; `replacements` is the extra
    keyword field) and `mol_from_smiles_with_params(smiles: &str, params:
    &SmilesParserParams) -> Result<Mol>`.
  - `SmilesWriteParams { all_bonds_explicit: bool, all_hs_explicit: bool,
    canonical: bool, clean_stereo: bool, do_isomeric_smiles: bool, do_kekule:
    bool, do_random: bool, ignore_atom_map_numbers: bool, include_dative_bonds:
    bool, rooted_at_atom: i32 }` and `mol_to_smiles_with_params(mol: &Mol,
    params: &SmilesWriteParams) -> Result<String>`.
  - `SmartsParserParams` (fields from the inventory); `mol_from_smarts(smarts:
    &str) -> Result<Mol>`; `mol_from_smarts_with_params`; `mol_to_smarts(mol:
    &Mol) -> Result<String>`; `mol_to_smarts_with_params(mol: &Mol, params:
    &SmilesWriteParams) -> Result<String>`.
  - `MolFromMolBlockParams` and `MolFromMolFileParams`, each `{ sanitize: true,
    remove_hs: true, strict_parsing: true }`; `mol_from_mol_block(mol_block:
    &str) -> Result<Mol>`; `mol_from_mol_file(mol_file_name: impl AsRef<Path>)
    -> Result<Mol>`; and their `_with_params` forms.
  - `MolToMolBlockParams` and `MolToMolFileParams`, each `{ include_stereo:
    true, conf_id: -1, kekulize: true, force_v3000: false }`;
    `mol_to_mol_block(mol: &Mol) -> Result<String>`; `mol_to_mol_file(mol:
    &Mol, filename: impl AsRef<Path>) -> Result<()>`; and their `_with_params`
    forms.
  - Paths cross the bridge as bytes (`as_encoded_bytes`), never as `&str`.

- [ ] **Step 1: Write the failing tests** in `tests/molfiles.rs`. Files go in
  `std::env::temp_dir()`, under names containing the process id.

  ```rust
  use rdkit::Error;
  use rdkit::chem::*;

  #[test]
  fn parser_and_writer_defaults() {
      let p = SmilesParserParams::default();
      assert!(p.allow_cxsmiles && p.parse_name && p.remove_hs && p.sanitize && p.strict_cxsmiles);
      assert_eq!((p.debug_parse, p.replacements.len()), (0, 0));
      let w = SmilesWriteParams::default();
      assert!(w.canonical && w.clean_stereo && w.do_isomeric_smiles && w.include_dative_bonds);
      assert!(!(w.all_bonds_explicit || w.all_hs_explicit || w.do_kekule || w.do_random || w.ignore_atom_map_numbers));
      assert_eq!(w.rooted_at_atom, -1);
  }

  #[test]
  fn smiles_params() {
      let unsanitized = SmilesParserParams { sanitize: false, ..Default::default() };
      assert_eq!(mol_from_smiles_with_params("C(C)(C)(C)(C)C", &unsanitized).unwrap().get_num_atoms(), 6);
      let mut params = SmilesParserParams::default();
      params.replacements.insert("{X}".into(), "O".into());
      assert_eq!(mol_to_smiles(&mol_from_smiles_with_params("C{X}", &params).unwrap()).unwrap(), "CO");
      let kekule = SmilesWriteParams { do_kekule: true, ..Default::default() };
      let benzene = mol_from_smiles("c1ccccc1").unwrap();
      assert_eq!(mol_to_smiles_with_params(&benzene, &kekule).unwrap(), "C1=CC=CC=C1");
  }

  #[test]
  fn smarts() {
      assert_eq!(mol_to_smarts(&mol_from_smarts("[OX2H]").unwrap()).unwrap(), "[O&X2&H1]");
      assert!(matches!(mol_from_smarts("[C"), Err(Error::Parse { .. })));
  }

  #[test]
  fn mol_blocks_and_files() {
      let mol = mol_from_smiles("CO").unwrap();
      let block = mol_to_mol_block(&mol).unwrap();
      assert_eq!(mol_to_smiles(&mol_from_mol_block(&block).unwrap()).unwrap(), "CO");
      assert!(matches!(mol_from_mol_block("garbage"), Err(Error::Parse { .. })));
      let path = std::env::temp_dir().join(format!("rdkit-rust-{}-co.mol", std::process::id()));
      mol_to_mol_file(&mol, &path).unwrap();
      assert_eq!(mol_to_smiles(&mol_from_mol_file(&path).unwrap()).unwrap(), "CO");
      assert!(matches!(mol_from_mol_file("does-not-exist.mol"),
                       Err(Error::Os { message }) if message.contains("does-not-exist.mol")));
  }

  #[test]
  fn latin1_title_is_a_value_error_not_an_abort() {
      let block = mol_to_mol_block(&mol_from_smiles("CO").unwrap()).unwrap();
      let rest = block.split_once('\n').unwrap().1;
      let path = std::env::temp_dir().join(format!("rdkit-rust-{}-latin1.mol", std::process::id()));
      std::fs::write(&path, [b"\xe9t\xe9\n".as_slice(), rest.as_bytes()].concat()).unwrap();
      let mol = mol_from_mol_file(&path).unwrap();
      assert!(matches!(mol_to_mol_block(&mol), Err(Error::Value { .. })));
  }
  ```

  Run: `cargo test --test molfiles`
  Expected: FAIL to compile.

- [ ] **Step 2: Implement it.** Mirrors of Python classes take their defaults
  from a wrapper function that returns a default-constructed C++ struct (A3).
  `<Name>Params` structs defined here write Python's defaults in Rust.

- [ ] **Step 3: Run the tests.**
  Expected: PASS.

- [ ] **Step 4: Update the manifest.** Remove the `MolFromSmiles`
  `overloads_note`, add this task's names, regenerate the paths, and commit:
  `git commit -m "Add SMARTS and molfile I/O with parameter structs"`.

### Task 8: Properties

**Files:**

- Create: `wrapper/include/rdkit-rust/props.h`, `wrapper/src/props.cpp`,
  `src/ffi/props.rs`, `src/chem/rdchem/props.rs`
- Modify: `src/chem/rdchem/{mol,atom,bond}.rs`, `CONVENTIONS.md`, manifest
- Test: `tests/props.rs`

**Interfaces:**

- Consumes: the views from Tasks 5 and 6.
- Produces:
  - `pub enum PropValue { Int(i32), Unsigned(u32), Double(f64), Bool(bool),
    String(String) }` (`Clone`, `Debug`, `PartialEq`), extended to whatever else the nanobind
    `GetPropsAsDict` converts.
  - `GetPropNamesParams { include_private: false, include_computed: false }`
    and `GetPropsAsDictParams { include_private: false, include_computed:
    false, auto_convert_strings: true }`.
  - Macros `prop_getters!` and `prop_setters!`, stamped on `Mol`, `Atom`,
    `AtomRef`, `AtomMut`, `BondRef`, and `BondMut` (getters), and on `Mol`,
    `Atom`, `AtomMut`, and `BondMut` (setters). Task 9 adds the conformer types.
  - Getters: `has_prop(key: &str) -> bool`, `get_prop(key) -> Result<String>`,
    `get_prop_auto_convert(key) -> Result<PropValue>` (A4),
    `get_int_prop(key) -> Result<i32>`, `get_unsigned_prop(key) ->
    Result<u32>`, `get_double_prop(key) -> Result<f64>`,
    `get_bool_prop(key) -> Result<bool>`, `get_prop_names() ->
    Result<Vec<String>>`, `get_prop_names_with_params(&GetPropNamesParams)`,
    `get_props_as_dict() -> Result<BTreeMap<String, PropValue>>`, and
    `get_props_as_dict_with_params(&GetPropsAsDictParams)`.
  - Setters: `set_prop`, `set_int_prop`, `set_unsigned_prop`,
    `set_double_prop`, and `set_bool_prop` (`key: &str`, then the value), each
    with `_with_computed(key, value, computed: bool)`; `clear_prop(key)`; and
    `clear_computed_props()`, plus the `_with_` forms A2 requires for the
    inventory's signatures.
  - One C++ template per operation in `props.h`, instantiated once per RDKit
    class in `props.cpp`. Type mismatches throw `ValueErrorException` with the
    nanobind wrapper's message.

- [ ] **Step 1: Write the failing tests** in `tests/props.rs`:

  ```rust
  mod common;

  use std::collections::BTreeMap;
  use rdkit::Error;
  use rdkit::chem::*;

  #[test]
  fn typed_getters_and_errors() {
      let mut mol = mol_from_smiles("OCC").unwrap();
      mol.set_prop("s", "abc");
      mol.set_int_prop("i", 7);
      assert_eq!(mol.get_prop("nope").unwrap_err(), Error::Key { key: "nope".into() });
      assert!(matches!(mol.get_int_prop("s"), Err(Error::Value { message }) if message.contains("integer")));
      assert_eq!(mol.get_prop("i").unwrap(), "7");
      assert!(matches!(mol.get_double_prop("i"), Err(Error::Value { .. })));
      mol.clear_prop("nope");
      assert_eq!(mol.get_prop_names().unwrap(), ["s", "i"]);
  }

  #[test]
  fn writing_smiles_sets_computed_properties() {
      let mol = mol_from_smiles("OCC").unwrap();
      mol_to_smiles(&mol).unwrap();
      let all = GetPropNamesParams { include_private: true, include_computed: true };
      assert!(mol.get_prop_names_with_params(&all).unwrap().contains(&"_smilesAtomOutputOrder".to_string()));
  }

  #[test]
  fn props_as_dict_converts_strings() {
      let mut mol = mol_from_smiles("C").unwrap();
      mol.set_prop("s", "abc");
      mol.set_int_prop("i", 7);
      mol.set_prop("n", "5");
      let expected = BTreeMap::from([
          ("i".to_string(), PropValue::Int(7)),
          ("n".to_string(), PropValue::Int(5)),
          ("s".to_string(), PropValue::String("abc".into())),
      ]);
      assert_eq!(mol.get_props_as_dict().unwrap(), expected);
  }

  #[test]
  fn atom_and_bond_props() {
      let mut atom = Atom::from_num(6);
      atom.set_prop("x", "y");
      let mut rw = RWMol::default();
      rw.add_atom(&atom);
      assert_eq!(rw.get_atom_with_idx(0).unwrap().get_prop("x").unwrap(), "y");
      let mut mol = mol_from_smiles("CC").unwrap();
      mol.get_bond_with_idx_mut(0).unwrap().set_double_prop("w", 0.5);
      assert_eq!(mol.get_bond_with_idx(0).unwrap().get_double_prop("w").unwrap(), 0.5);
  }

  #[test]
  fn latin1_name_is_a_value_error() {
      let mol = mol_from_mol_file(common::latin1_mol_file()).unwrap();
      assert!(matches!(mol.get_prop("_Name"), Err(Error::Value { .. })));
  }
  ```

  Move Task 7's Latin-1 file builder into `tests/common/mod.rs` as
  `pub fn latin1_mol_file() -> PathBuf`, and use it from both files.
  Run: `cargo test --test props`
  Expected: FAIL to compile.

- [ ] **Step 2: Implement it.**

- [ ] **Step 3: Run the tests.**
  Expected: PASS.

- [ ] **Step 4: Add rule P1 to `CONVENTIONS.md`:** properties follow A2 and
  A4 like any method. Missing keys are `Error::Key`, type mismatches and
  invalid UTF-8 are `Error::Value`, and the getters and setters come from the
  props macros rather than being written per class.

- [ ] **Step 5: Update the manifest**, regenerate the paths, and commit:
  `git commit -m "Add typed properties"`.

### Task 9: Conformers, points, and ring info

**Files:**

- Create: `wrapper/include/rdkit-rust/{conformer,ring_info}.h`,
  `wrapper/src/{conformer,ring_info}.cpp`, `src/ffi/{conformer,ring_info}.rs`,
  `src/geometry/mod.rs`, `src/geometry/rd_geometry.rs`,
  `src/chem/rdchem/{conformer,ring_info}.rs`
- Modify: `src/lib.rs`, `src/chem/rdchem/mol.rs`, `CONVENTIONS.md`, manifest
- Test: `tests/conformers.rs`, `tests/ring_info.rs`

**Interfaces:**

- Consumes: the props macros, `Error`.
- Produces:
  - `rdkit::geometry::rd_geometry::Point3D { pub x: f64, pub y: f64, pub z:
    f64 }` (a cxx shared struct: `Copy`, `PartialEq`, `Debug`, `Default`) with
    `new(x: f64, y: f64, z: f64)`.
  - `Conformer` (owned: `Default`, `new(num_atoms: u32)`, `Clone`),
    `ConformerRef<'m>`, and `ConformerMut<'m>`. Getters: `get_id() -> u32`,
    `get_num_atoms() -> u32`, `is_3d() -> bool` (`name_exception` for `Is3D`),
    `get_atom_position(aid: u32) -> Result<Point3D>`, and
    `get_positions() -> Vec<Point3D>`; plus `get_owning_mol` on views.
    Setters: `set_atom_position(aid: u32, loc: Point3D) -> Result<()>`,
    `set_3d(v: bool)` (`name_exception`), `set_id(id: u32)`. Props on all
    three.
  - On `Mol`: `get_num_conformers() -> u32`, `get_conformer() ->
    Result<ConformerRef<'_>>`, `get_conformer_with_id(id: i32)`,
    `get_conformer_mut()`, `get_conformer_with_id_mut(id: i32)`,
    `get_conformers() -> Conformers<'_>`, `add_conformer(&mut self, conf:
    &Conformer) -> Result<u32>`, `add_conformer_with_assign_id(&mut self, conf:
    &Conformer, assign_id: bool) -> Result<u32>`, `remove_conformer(&mut self,
    id: u32)`, `remove_all_conformers(&mut self)`, and `get_ring_info(&self) ->
    RingInfoRef<'_>`.
  - `RingInfoRef<'m>`: `num_rings() -> Result<u32>`, `atom_rings() ->
    Result<Vec<Vec<u32>>>`, `bond_rings() -> Result<Vec<Vec<u32>>>`,
    `num_atom_rings(idx: u32) -> Result<u32>`, `num_bond_rings(idx: u32) ->
    Result<u32>`, `is_atom_in_ring_of_size(idx: u32, size: u32) ->
    Result<bool>`, and `is_bond_in_ring_of_size(idx: u32, size: u32) ->
    Result<bool>`.

- [ ] **Step 1: Write the failing tests.** `tests/conformers.rs`:

  ```rust
  use rdkit::Error;
  use rdkit::chem::*;
  use rdkit::geometry::rd_geometry::Point3D;

  #[test]
  fn missing_and_added_conformers() {
      let mut mol = mol_from_smiles("CO").unwrap();
      assert_eq!(mol.get_num_conformers(), 0);
      assert!(matches!(mol.get_conformer(), Err(Error::Value { message }) if message.contains("Bad Conformer Id")));
      let mut conf = Conformer::new(2);
      conf.set_atom_position(1, Point3D::new(1.5, 0.0, 0.0)).unwrap();
      assert_eq!(mol.add_conformer_with_assign_id(&conf, true).unwrap(), 0);
      let got = mol.get_conformer().unwrap();
      assert_eq!(got.get_atom_position(1).unwrap(), Point3D::new(1.5, 0.0, 0.0));
      assert!(got.is_3d());
      assert_eq!(got.get_positions().len(), 2);
      assert_eq!(got.get_atom_position(5).unwrap_err(), Error::Index { index: 5, len: 2 });
  }

  #[test]
  fn conformer_must_match_atom_count() {
      let mut mol = mol_from_smiles("CO").unwrap();
      assert!(matches!(mol.add_conformer(&Conformer::new(3)),
                       Err(Error::Runtime { message }) if message.contains("mismatch")));
  }

  #[test]
  fn edit_through_conformer_mut() {
      let mut mol = mol_from_mol_block(&mol_to_mol_block(&mol_from_smiles("CO").unwrap()).unwrap()).unwrap();
      assert_eq!(mol.get_num_conformers(), 1);
      mol.get_conformer_mut().unwrap().set_atom_position(0, Point3D::new(0.0, 2.0, 0.0)).unwrap();
      assert_eq!(mol.get_conformer().unwrap().get_atom_position(0).unwrap().y, 2.0);
  }
  ```

  `tests/ring_info.rs`:

  ```rust
  use std::collections::BTreeSet;
  use rdkit::Error;
  use rdkit::chem::*;

  #[test]
  fn ring_queries() {
      let mol = mol_from_smiles("c1ccccc1C1CC1").unwrap();
      let info = mol.get_ring_info();
      assert_eq!(info.num_rings().unwrap(), 2);
      let rings: BTreeSet<BTreeSet<u32>> =
          info.atom_rings().unwrap().into_iter().map(|r| r.into_iter().collect()).collect();
      assert_eq!(rings, BTreeSet::from([BTreeSet::from([0, 1, 2, 3, 4, 5]), BTreeSet::from([6, 7, 8])]));
      assert_eq!(info.num_atom_rings(6).unwrap(), 1);
      assert!(info.is_atom_in_ring_of_size(6, 3).unwrap());
      assert!(info.is_atom_in_ring_of_size(0, 6).unwrap());
      assert_eq!(info.num_atom_rings(99).unwrap_err(), Error::Index { index: 99, len: 9 });
  }

  #[test]
  fn uninitialized_ring_info_is_an_error_not_an_abort() {
      let params = SmilesParserParams { sanitize: false, ..Default::default() };
      let mol = mol_from_smiles_with_params("C1CC1", &params).unwrap();
      assert!(matches!(mol.get_ring_info().num_rings(),
                       Err(Error::Runtime { message }) if message.contains("RingInfo not initialized")));
  }
  ```

  Add a `compile_fail` doctest showing a `RingInfoRef` cannot outlive its `Mol`.
  Run: `cargo test --test conformers --test ring_info`
  Expected: FAIL to compile.

- [ ] **Step 2: Implement it.**

- [ ] **Step 3: Run the tests.**
  Expected: PASS.

- [ ] **Step 4: Update the manifest** (`Point3D.x`, `.y`, and `.z` with
  `rust_kind = "field"`), regenerate the paths, and commit:
  `git commit -m "Add conformers, points, and ring info"`.

### Task 10: Sanitization and molecule operations

**Files:**

- Create: `wrapper/include/rdkit-rust/{molops,mol_list}.h`,
  `wrapper/src/{molops,mol_list}.cpp`, `src/ffi/molops.rs`,
  `src/chem/rdmolops.rs`
- Modify: `src/chem/mod.rs`, `CONVENTIONS.md`, manifest
- Test: `tests/molops.rs`

**Interfaces:**

- Consumes: `Mol::pin_mut`, `Mol::from_ffi`, `SmilesWriteParams`.
- Produces in `rdkit::chem::rdmolops`:
  - `SanitizeFlags(u32)`, with `Copy`, `Eq`, `Debug`, `BitOr`, `BitAnd`, `Not`,
    `BitOrAssign`, `bits()`, and `from_bits_retain(u32)`, and constants with
    `MolOps.h`'s values: `SANITIZE_NONE` 0x0, `SANITIZE_CLEANUP` 0x1,
    `SANITIZE_PROPERTIES` 0x2, `SANITIZE_SYMMRINGS` 0x4, `SANITIZE_KEKULIZE`
    0x8, `SANITIZE_FINDRADICALS` 0x10, `SANITIZE_SETAROMATICITY` 0x20,
    `SANITIZE_SETCONJUGATION` 0x40, `SANITIZE_SETHYBRIDIZATION` 0x80,
    `SANITIZE_CLEANUPCHIRALITY` 0x100, `SANITIZE_ADJUSTHS` 0x200,
    `SANITIZE_CLEANUP_ORGANOMETALLICS` 0x400, `SANITIZE_CLEANUPATROPISOMERS`
    0x800, and `SANITIZE_ALL` 0xFFFFFFF.
  - `sanitize_mol(mol: &mut Mol) -> Result<SanitizeFlags>`;
    `SanitizeMolParams { sanitize_ops: SANITIZE_ALL, catch_errors: false }`;
    and `sanitize_mol_with_params(mol: &mut Mol, params: &SanitizeMolParams) ->
    Result<SanitizeFlags>`. Like Python, these return the failed operation when
    `catch_errors` is set, and `SANITIZE_NONE` on success.
  - `kekulize(mol: &mut Mol) -> Result<()>` and
    `kekulize_with_clear_aromatic_flags(mol: &mut Mol, clear_aromatic_flags:
    bool) -> Result<()>`.
  - `AddHsParams { explicit_only: false, add_coords: false, only_on_atoms:
    Option<Vec<u32>> = None, add_residue_info: false }`; `add_hs(mol: &Mol) ->
    Result<Mol>`; and `add_hs_with_params(mol: &Mol, params: &AddHsParams) ->
    Result<Mol>`.
  - `get_mol_frags(mol: &Mol) -> Vec<Vec<u32>>`, `get_mol_frags_as_mols(mol:
    &Mol) -> Result<Vec<Mol>>`, and
    `get_mol_frags_as_mols_with_sanitize_frags(mol: &Mol, sanitize_frags: bool)
    -> Result<Vec<Mol>>`. `frags` and `fragsMolAtomMapping` are Python output
    arguments (A4).
  - C++ `rdkit_rust::MolList` (`std::size_t size() const;
    std::unique_ptr<RDKit::RWMol> take(std::size_t i);`), and
    `pub(crate) fn mol_list_into_vec(list: UniquePtr<ffi::MolList>) ->
    Vec<Mol>`. Each `ROMOL_SPTR` is copied into a new `RWMol`; RDKit 2026.09
    has no `RWMol(ROMol&&)`.

- [ ] **Step 1: Write the failing tests** in `tests/molops.rs`:

  ```rust
  use rdkit::Error;
  use rdkit::chem::*;

  fn unsanitized(smiles: &str) -> Mol {
      mol_from_smiles_with_params(smiles, &SmilesParserParams { sanitize: false, ..Default::default() }).unwrap()
  }

  #[test]
  fn sanitize() {
      assert_eq!(sanitize_mol(&mut unsanitized("CCO")).unwrap(), SanitizeFlags::SANITIZE_NONE);
      assert!(matches!(sanitize_mol(&mut unsanitized("C(C)(C)(C)(C)C")),
                       Err(Error::AtomValence { atom_idx: 0, .. })));
      let catch = SanitizeMolParams { catch_errors: true, ..Default::default() };
      assert_eq!(sanitize_mol_with_params(&mut unsanitized("C(C)(C)(C)(C)C"), &catch).unwrap(),
                 SanitizeFlags::SANITIZE_PROPERTIES);
      let skip = SanitizeMolParams {
          sanitize_ops: SanitizeFlags::SANITIZE_ALL & !SanitizeFlags::SANITIZE_PROPERTIES,
          ..Default::default()
      };
      assert_eq!(sanitize_mol_with_params(&mut unsanitized("C(C)(C)(C)(C)C"), &skip).unwrap(),
                 SanitizeFlags::SANITIZE_NONE);
      assert_eq!(SanitizeFlags::SANITIZE_ALL.bits(), 0xFFFFFFF);
      assert_eq!((SanitizeFlags::SANITIZE_KEKULIZE | SanitizeFlags::SANITIZE_CLEANUP).bits(), 0x9);
  }

  #[test]
  fn kekulize_forms() {
      let mut benzene = mol_from_smiles("c1ccccc1").unwrap();
      kekulize_with_clear_aromatic_flags(&mut benzene, true).unwrap();
      assert!(benzene.get_atoms().all(|a| !a.get_is_aromatic()));
      let kekule = SmilesWriteParams { do_kekule: true, ..Default::default() };
      assert_eq!(mol_to_smiles_with_params(&benzene, &kekule).unwrap(), "C1=CC=CC=C1");
      assert!(matches!(kekulize(&mut unsanitized("c1cccc1")),
                       Err(Error::Kekulize { atom_indices, .. }) if atom_indices == [0, 1, 2, 3, 4]));
  }

  #[test]
  fn add_hs_returns_a_new_molecule() {
      let methane = mol_from_smiles("C").unwrap();
      assert_eq!(add_hs(&methane).unwrap().get_num_atoms(), 5);
      assert_eq!(methane.get_num_atoms(), 1);
      let only_o = AddHsParams { only_on_atoms: Some(vec![1]), ..Default::default() };
      assert_eq!(add_hs_with_params(&mol_from_smiles("CO").unwrap(), &only_o).unwrap().get_num_atoms(), 3);
  }

  #[test]
  fn fragments() {
      let mol = mol_from_smiles("CC.O").unwrap();
      assert_eq!(get_mol_frags(&mol), [vec![0, 1], vec![2]]);
      let frags: Vec<_> = get_mol_frags_as_mols(&mol).unwrap().iter().map(|m| mol_to_smiles(m).unwrap()).collect();
      assert_eq!(frags, ["CC", "O"]);
      assert!(get_mol_frags(&mol_from_smiles("").unwrap()).is_empty());
  }
  ```

  Run: `cargo test --test molops`
  Expected: FAIL to compile.

- [ ] **Step 2: Implement it.** In-place operations pass the C++ `RWMol&`
  (W1). `add_hs` copies its input into a new `RWMol` and calls RDKit's
  in-place `addHs`.

- [ ] **Step 3: Run the tests.**
  Expected: PASS.

- [ ] **Step 4: Add rules to `CONVENTIONS.md`:**
  - **W6.** Functions that return new molecules copy into an `RWMol`; vectors
    of molecules cross as `MolList`.
  - **F1.** Python flag enums become a `u32` newtype with associated
    constants, bit operators, `bits()`, and `from_bits_retain`.

- [ ] **Step 5: Update the manifest**, regenerate the paths, and commit:
  `git commit -m "Add sanitization, kekulization, Hs, and fragments"`.

### Task 11: Substructure matching

**Files:**

- Create: `wrapper/include/rdkit-rust/substruct.h`,
  `wrapper/src/substruct.cpp`, `src/ffi/substruct.rs`,
  `src/chem/rdchem/substruct.rs`
- Modify: `wrapper/CMakeLists.txt` (target `SubstructMatch`), manifest
- Test: `tests/substruct.rs`

**Interfaces:**

- Consumes: `Mol`, `mol_from_smarts`, `SmilesParserParams`.
- Produces in `rdkit::chem::rdchem`:
  - `SubstructMatchParameters`, the A3 mirror of the Python class, with
    `Default` from C++ and fields from the inventory: among them `uniquify`
    (true), `use_chirality` (false), `use_query_query_matches` (false),
    `recursion_possible` (true), `max_matches: u32` (1000),
    `max_recursive_matches: u32` (1000), `num_threads: i32` (1),
    `atom_properties: Vec<String>`, and `bond_properties: Vec<String>`.
  - Methods on `Mol`: `has_substruct_match(&self, query: &Mol) -> bool`,
    `get_substruct_match(&self, query: &Mol) -> Vec<u32>`,
    `get_substruct_matches(&self, query: &Mol) -> Vec<Vec<u32>>`, and a
    `_with_params(&self, query: &Mol, params: &SubstructMatchParameters)` form
    of each. The `MolBundle` overloads get
    `overloads_note = "MolBundle is not bound yet"`.

- [ ] **Step 1: Write the failing tests** in `tests/substruct.rs`:

  ```rust
  use rdkit::chem::*;

  #[test]
  fn matches() {
      let catechol = mol_from_smiles("Oc1ccccc1O").unwrap();
      let hydroxyl = mol_from_smarts("[OX2H]").unwrap();
      assert!(catechol.has_substruct_match(&hydroxyl));
      assert_eq!(catechol.get_substruct_match(&hydroxyl), [0]);
      assert_eq!(catechol.get_substruct_matches(&hydroxyl), [vec![0], vec![7]]);
      let amine = mol_from_smarts("N").unwrap();
      assert!(!catechol.has_substruct_match(&amine));
      assert!(catechol.get_substruct_match(&amine).is_empty());
      let one = SubstructMatchParameters { max_matches: 1, ..Default::default() };
      assert_eq!(catechol.get_substruct_matches_with_params(&hydroxyl, &one).len(), 1);
  }

  #[test]
  fn chirality_is_opt_in() {
      let target = mol_from_smiles("C[C@H](F)Cl").unwrap();
      let query = mol_from_smiles("C[C@@H](F)Cl").unwrap();
      assert!(target.has_substruct_match(&query));
      let chiral = SubstructMatchParameters { use_chirality: true, ..Default::default() };
      assert!(!target.has_substruct_match_with_params(&query, &chiral));
  }

  #[test]
  fn ring_query_on_an_unsanitized_target() {
      let params = SmilesParserParams { sanitize: false, ..Default::default() };
      let target = mol_from_smiles_with_params("C1CC1", &params).unwrap();
      assert!(target.has_substruct_match(&mol_from_smarts("[R]").unwrap()));
  }
  ```

  Run: `cargo test --test substruct`
  Expected: FAIL to compile.

- [ ] **Step 2: Implement it.** Where the nanobind methods do more than call
  `SubstructMatch` (ring-info setup on unsanitized molecules, for one),
  the C++ wrapper does the same.

- [ ] **Step 3: Run the tests.**
  Expected: PASS.

- [ ] **Step 4: Update the manifest**, regenerate the paths, and commit:
  `git commit -m "Add substructure matching"`.

- [ ] **Step 5: Close the core slice.** Check that `CONVENTIONS.md` has one
  settled rule for each pattern the design lists: ownership and views (W1–W5),
  `RWMol` and in-place operations (W1, W6), new molecules and vectors (W6),
  parameter structs (A2, A3), overload naming (A5), errors (E1–E5), iterators
  (W5), typed properties (P1), and `Send`/`Sync` (S1). Push `core-slice` and
  update the draft PR's description.
