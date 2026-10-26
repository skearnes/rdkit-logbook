# RDKit bugs found while writing rdkit-cxx

- **Date:** 2026-10-26
- **Author:** Steven Kearnes
- **Acknowledgments:** Prepared with [Claude Code](https://claude.com/claude-code)
- **License:** [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/)

These are defects in RDKit `Release_2026_09_1` found while writing
[rdkit-cxx](https://github.com/skearnes/rdkit-cxx). Most are reachable from
RDKit's Python wrappers. None has been reported upstream yet. The crate guards
against each one with a check, a lock, or by leaving the name unbound, as
described in each entry.

This page collects what was recorded while binding, during the pre-release
review, and, in the last section, while porting RDKit's Python modules. The
wrapper's comments are the full record: every check in `wrapper/src` says what RDKit would do without
it, and every unbound name's entry in `inventory/manifest.toml` gives the
reason and, where there is one, the crash in Python that confirmed it. Many
smaller cases appear only there, such as RDKit trusting an index it is given or
a size it computes in `int`.

## Memory safety

- **`RWMol::replaceBond` leaves stereo groups pointing at the freed bond**
  (`Code/GraphMol/RWMol.cpp`). It deletes the bond it replaces but, unlike
  `replaceAtom`, does not update the stereo groups that hold it, such as an
  atropisomer's. Copying, pickling, or writing the molecule then reads freed
  memory. Memcheck confirmed this in Python through `ReplaceBond` followed by
  `ToBinary`, `AdjustQueryProperties` with generic bonds,
  `CreateExtendedQueryMol` with tautomers, `SanitizeRxn`, and
  `GenerateDepictionMatching2DStructure` with `allowRGroups`. *Crate:* its own
  calls point the groups at the new bond, and RDKit's own calls run under a
  guard that takes the bonds out of the groups and puts them back afterward
  (rule U13).
- **`EHTTools::runMol` reads `FILE`s it already closed**
  (`External/YAeHMOP/EHTTools.cpp`). It `fclose`s its two temporary files,
  then the `Tempfile` destructors call `fileno` and `fclose` on them again.
  That is a use after free on every call (Valgrind reports the reads in
  Python), and a double free once another thread reuses the descriptor.
  *Crate:* `rdEHTTools` is unbound.
- **InChI's own mol-file reader trusts its input.** `MolBlockToInchi` parses
  with the InChI library's C reader (`MakeINCHIFromMolfileText`), not RDKit's,
  and it segfaulted on 31 of 5,800 mutated V3000 blocks in Python (26 of them
  reproduce alone). *Crate:* unbound; parse with `mol_from_mol_block` and call
  `mol_to_inchi`.
- **InChI options that corrupt the heap.** With `/OutputSDF /Key`,
  `GetINCHI` `free()`s a pointer it never allocated (SIGABRT in Python).
  *Crate:* only the 22 structure-perception options are accepted.
- **The Avalon toolkit trusts its input** (`External/AvalonTools`, about
  34,000 lines of C). Malformed SMILES such as `ClC(Cl)=C1=CC=CC..1` corrupt
  its heap, and malformed mol blocks are read out of bounds (both crash
  Python). Its layout and fingerprints go on after `SetupNeighbourhood` fails
  for an atom of more than 20 bonds, having already written past that atom's
  neighbor table; a sanitized molecule can have such an atom (a dummy atom with
  24 neighbors). `nBits` 0 divides by zero. The structure checker reads through
  a null `FILE*` when a file its options name does not open (`-ta /nonexistent`
  segfaults). *Crate:* `pyAvalonTools` is unbound.
- **`MaeMolSupplier` trusts maeparser's parse.** It dereferences a missing
  bond-order key's end iterator, subtracts 1 from `INT_MIN`, and takes `back()`
  of an empty token. *Crate:* unbound, since maeparser's headers, which a check
  would need, are not installed.
- **`FreeChemicalFeature::initFromString` reads past its buffers**
  (`Code/ChemicalFeatures/FreeChemicalFeature.cpp`). It copies each string into
  a buffer of the length the pickle gives, without checking how much it read,
  then takes the string up to the first NUL. With no NUL, or a pickle that ends
  early, that reads past the buffer or through bytes it never read. It also
  throws a `const char *` for an unknown version, which no `std::exception`
  handler catches. *Crate:* checks the pickle first.
- **The JSON reader trusts indices and types**
  (`Code/GraphMol/MolInterchange/Parser.cpp`). An AND, OR, or XOR query with
  fewer than two children segfaults, a recursive query's matcher reads the
  first atom of an empty subquery, and ring atoms can be out of range.
  *Crate:* checks the JSON first.
- **Abbreviations.** `applyMatches` overflows the bitset it marks atoms in, and
  `parseAbbreviations` runs its token iterator past the end. *Crate:* checks.
- **Enumeration.** Link nodes read `ps[0][0]` without checking that it exists
  (the `ASSERT_INVARIANT` is compiled out), and repeat units use SRU indices
  unchecked. *Crate:* checks as RDKit reads them.
- **Synthon spaces.** A text read in place that fails leaves synthons without
  molecules, and a later search segfaults. Negative synthon numbers, more than
  four synthon sets, and an empty or all-dummy synthon (index -1) are
  accepted. A search's possible hits dereference null for SMILES that do not
  parse. *Crate:* checks.
- **Drawing.**
  - `addMetadataToPNG` compares the bytes of a PNG signature it failed to
    read, which are uninitialized, when the Cairo surface is in an error state.
  - The Cairo drawer draws free text to a null context.
  - `getDrawCoords` indexes `drawMols_[activeMolIdx_]` unchecked.
  - The JSON options parser (`get_rgba` in `MolDrawOptionsJSONParsers.cpp`)
    reads past the end of a color with fewer than three components.
  - `getWavyLineSegments` rounds an odd count up, which makes `UINT_MAX` zero
    segments, and the drawers then read the first one.

  *Crate:* checks each before drawing.
- **Picking.** `MaxMinPicker` dereferences null when no distance is above -1.
  *Crate:* checks.
- **Python wrappers.** `MolChemicalFeatureFactory.GetMolFeature` keeps a
  function-static feature list shared across threads, which leaves features
  dangling. `GetAtomMatch` indexes a bitset of `maxAts` bits without a bounds
  check (`Code/GraphMol/MolChemicalFeatures/Wrap`, and the nanobind port as
  well). `MurtaghCluster` reads `nPts * sz` values from its data without
  checking that the array holds them. In the trajectory module,
  `Trajectory.GetSnapshot` returns a copy that keeps a raw pointer to the
  trajectory, which dangles once the trajectory is gone, and a `Snapshot`
  with fewer coordinates than its trajectory's points is read past its end.
  *Crate:* its own bindings avoid each.
- **Molecule pickles RDKit did not write** (`Code/GraphMol/MolPickler.cpp`).
  `MolPickler` takes a query's class from its tag and its matching function
  from its description independently, and converts what it reads without
  range checks, so a pickle RDKit did not write reaches undefined behavior
  while it is read or later. One mutated byte in an `ExtendedQueryMol`'s JSON,
  which holds molecule pickles in base64, segfaulted Python. RDKit fuzzes the
  parsing of pickles, not what the parsed molecule then does. *Crate:* the
  pickle readers are `unsafe fn`s whose contract is a pickle RDKit wrote,
  and they keep the checks they have.
- **Eigen's solvers on coordinates that are not finite.** Eigen's SVD and
  eigensolvers leave their output uninitialized for input that is not finite,
  and RDKit reads it; memcheck showed it in Python for `WHIM`, `PBF`, and
  `GETAWAY`. `GETAWAY` and the EEM charges also divide by interatomic
  distances, which are 0 for atoms at the same place, as embedding
  `[Na+].[Cl-]` leaves them. *Crate:* refuses coordinates that are not finite
  or beyond 1e9, and, for those two, atoms within 1e-6 of each other.
- **InChI and tetrahedral centers with fewer than three neighbors.**
  `MolToInchi` passes InChI two uninitialized neighbor indices for such a
  center, such as `[C@H](F)Cl` from SMARTS. *Crate:* refuses one.
- **The property-table descriptors read outside their tables**
  (`Code/GraphMol/Descriptors/MolData3Ddescriptors.cpp`). `AUTOCORR2D`,
  `AUTOCORR3D`, `RDF`, `MORSE`, `WHIM`, and `GETAWAY` look up atomic
  properties by atomic number minus one in tables that cover 1 to 110, so a
  dummy atom reads before the start and an atom past darmstadtium past the
  end. In Python, `CalcAUTOCORR2D` of `CC*` returns values read from outside
  the table. *Crate:* refuses such atoms unless a custom property is given,
  so the descriptor calculators give NaN for them.
- **`ROMol*` cast to `RWMol*`.** `removeQueryAtoms` and
  `generateIsomerConformers` `reinterpret_cast` one to the other. That is
  formally undefined, though the layouts are compatible.

## Data races

Python's global interpreter lock hides most of these, but RDKit releases it
in many calls, and C++ and Rust callers have no such lock.

- **Every substructure match races on the Ctrl-C handler**
  (`Code/RDGeneral/ControlCHandler.h`). `vf2` constructs a `ControlCHandler`
  for every match, and its constructor and destructor write the plain statics
  `d_prev_handler` and `d_gotSignal`, which the match loop reads. Two threads
  matching at once race on them, which can leave RDKit's `SIGINT` handler
  installed, and a stale `d_gotSignal` stops later matches early. *Crate:*
  `control_c_mutex()` is held around every call that can match. A fix to RDKit
  is drafted on `skearnes/rdkit`, branch `thread-safe-controlc`: atomic
  statics, and a handler installed on the first holder and restored by the
  last.
- **The log flags** (`Code/RDGeneral/RDLog.h`). `df_enabled` is a plain `bool`
  that every `BOOST_LOG` reads. `RDLog::LogStateSetter` writes it inside RDKit
  (`hasQueryHs` in `AddHs.cpp`, `prepareMolForDrawing`, deprotection, R-group
  decomposition, link nodes, the embedder's torsion preferences), as do
  `BlockLogs` and `enable_logs`/`disable_logs`. Its destructor also restores
  the flags with an XOR of what its constructor changed, so setters that
  overlap on two threads can restore the wrong state. *Crate:*
  RDKit's own writers all run under `control_c_mutex()`, which orders them
  with each other, but the race between a writer and the logging in other
  threads is still open (see the report's Known limitations).
- **The ChemDraw library's attribute maps**
  (`InitAttributeMapsIfNeeded` in
  `External/ChemDraw/chemdraw/chemdraw/CDXObject.cpp`). The maps are filled on
  first use behind an `empty()` check made outside the lock, so a thread can
  read a map another is still filling. The map of unknown tags is written with
  no lock. This showed up in CI as wrong attribute names in CDXML (`attrib0805`
  for `BondLength`) and was confirmed under DRD. *Crate:* `chemdraw_mutex()`.
- **Murtagh's clustering** (`Code/ML/Cluster/Murtagh/hc.c`) keeps its work
  variables in statics. *Crate:* `murtagh_mutex()`.
- **Other process-wide state written without a lock:**
  - the grid-index cache in `GridUtils.cpp`;
  - the BRICS patterns, filled on first use;
  - the descriptor property registry;
  - `DCLV.cpp`'s `recordCache`;
  - R-group decomposition's random number singleton and fingerprint-variance
    static;
  - the shape modules' pharmacophore patterns and PubChem shape's overlap
    cutoffs;
  - 2D depiction's settings and CoordGen's templates;
  - the SMILES parser's debug level (`yysmiles_debug`);
  - `std::rand` in comic-mode drawing.

  *Crate:* a lock for each (rule U4). Calls that write the SMILES debug level
  are `unsafe fn`s. CI checks the locks under Valgrind's DRD.

## Undefined arithmetic and casts

- **Molecule pickles: unchecked casts** (`Code/GraphMol/MolPickler.cpp`).
  Enum values (bond type, direction, and stereo; chiral tag; hybridization;
  stereo-group type) are cast without checks. *Crate:* refuses unknown values
  after every read. In `_unpickleAtomData` an isotope is computed as
  `static_cast<int>(floor(tmpFloat + mass))` from a float in the pickle, which
  is undefined when the value does not fit in `int`. *Crate:* not guarded (see
  the report's Known limitations).
- **The ChemDraw writer's fixed-point coordinates.**
  `CDXCoordinatefromPoints` (`CDXIO.h`) C-casts each coordinate to 32-bit fixed
  point, which is undefined when the value is not finite or does not fit. In 2D
  it first scales by 28.8 over the mean bond length, which is NaN for a
  molecule without bonds and infinite when the bonds have no length. *Crate:*
  checks the values first.
- **Synthon-space progress bar.** `ProgressBar.h` converts its projected finish
  to integer nanoseconds with `duration_cast`, which overflows past 2^63 ns.
  *Crate:* the bar is not bound.
- **Synthon-space shape search.** `alignDummies` computes a sine as
  `sqrt(1 - cos^2)`, NaN when rounding puts the cosine past 1, and
  `GaussianShape::AlignShape` then returns its best score uninitialized.
  *Crate:* `ShapeSearch` is not bound.
- **Pickers.** `MaxMinPicker`'s work array wraps (unsigned) at 2^28 items.
  `getDistFromLTM` (`DistPicker.cpp`) wraps past 65,537 items. *Crate:*
  refuses those sizes.
- **Information theory** (`Code/ML/InfoTheory`). `ChiSquare` and
  `InfoEntropyGain` truncate the matrix's totals to `int`. `getTopN(0)`
  compares against the top of an empty queue. *Crate:* checks.
- **Division by zero.** `MetricMatrixCalc` divides by zero (SIGFPE in Python)
  for a fingerprint with no bits, and so does R-group decomposition for a chunk
  size of 0. *Crate:* checks.
- **Drawing.** `DrawShapeArc::myDraw` steps its angles in `int` (undefined for
  NaN), and the SVG drawer converts contour colors that an infinite grid value
  makes NaN to `int`. *Crate:* checks.
- **Conformer ids.** `addConformer` with `assignId` gives the new conformer
  `maxId + 1` in `int`, which overflows when a conformer's id is `INT_MAX`.
  *Crate:* refuses that, and ids above `i32::MAX`.
- **Trajectory readers.** `readAmberTrajectory` and `readGromosTrajectory`
  size each frame as `numPoints * 3` in `unsigned int`, which wraps for large
  point counts, and then write `numPoints * 3` coordinates into the smaller
  array. *Crate:* refuses such counts.
- **`DistGeom.DoTriangleSmoothing`'s Python binding** computes
  `nrows * nrows` in `int`, which overflows above 46,340 rows. *Crate:*
  checks the size.
- **The InChI writer narrows silently** (`External/INCHI-API/inchi.cpp`).
  Molecules of more than 32,766 atoms return an empty string, an isotope of
  70,000 becomes a wrong `/i` layer, and 256 explicit hydrogens become `CH4`.
  *Crate:* refuses atoms, isotopes, and hydrogen counts that do not fit.

## Hangs and stack overflows

- **Substance-group `PARENT` cycles.** Whenever RDKit removes atoms or bonds,
  it follows the substance groups' `PARENT` chains with no check for a cycle,
  and it parses every bond's `_MolFileBondEndPts` without checks. The SMILES
  writer removes atoms from copies to split fragments, so `MolToSmiles` of a
  two-fragment molecule with a `PARENT` cycle hangs in Python; a CXSMILES `SgH`
  cycle alone does it. *Crate:* every wrapper whose RDKit call removes atoms or
  bonds checks first (rule P2).
- **Comic-mode drawing.** `handdrawnLine` (`MolDraw2DDetails.cpp`) shrinks its
  wiggles until they are small next to the line's steps, which never ends for a
  step of no length. In a comic-mode reaction on a small canvas, the plus
  signs' strokes (`plusWidth / 3`, in `int`) and the arrow's shaft can have
  no length. *Crate:* refuses such lines.
- **Feature definitions without `EndFeature`.** `parseFeatureDef`
  (`Code/GraphMol/MolChemicalFeatures/FeatureParser.cpp`) rereads its last
  line forever when a definition runs to the end of its input. *Crate:*
  refuses such a definition.
- **AMBER trajectories of no points.** `readAmberTrajectory` adds empty
  snapshots forever, without reading on, for a trajectory of 0 points once
  the file's title line ends in a newline. *Crate:* refuses 0 points.
- **Atom labels.** `AtomSymbol::adjustColons` strips `<...>` in a loop that
  never ends once a `>` comes before the next `<`. *Crate:* checks labels.
- **SMILES files whose stream reports its position but cannot seek.**
  `SmilesMolSupplier` loops forever after a failed seek, since `skipComments`
  stops only at end of file or `badbit`. *Crate:* reads gzipped files through
  a stream that fails every seek and tell, as RDKit's own `gzstream` does.
- **SLN macros.** `replaceSLNMacroAtoms` (`SLNParse.cpp`) runs `std::regex`
  `\{(.+?):(.+?)\}` over its input, and libstdc++'s recursive executor
  overflows the stack when a `{` is followed by about 100,000 characters (a
  segfault in Python). *Crate:* documented as an abort.
- **Deep SMILES.** `MolToSmiles` overflows a 2 MiB thread stack on a chain of
  about 16,000 atoms.
- **`LeaderPicker` threads.** `pthread_create`'s result is unchecked, and an
  exception in a worker thread calls `std::terminate`. *Crate:* runs the picks
  on the calling thread.

## Unbounded allocation

- **CDXML reaction atom maps.** One `ReactionStepAtomMap` token that is not an
  `int`, such as `x` or `4294967295`, makes the ChemDraw library append until
  memory runs out. *Crate:* refuses such a token.
- **InChI isotopic hydrogens.** `/i1D200` wraps to -56 in InChI's
  `signed char`; RDKit reads that as about 4.3 billion and adds that many
  atoms. *Crate:* refuses H, D, or T counts of 128 or more.
- **Real-valued vector and grid pickles.** `RealValueVect` and
  `UniformRealValueGrid3D` allocate the length a pickle claims before reading
  it, up to 32 GB from 8 bytes. *Crate:* refuses pickles that claim more
  than they hold.

## Wrong results

- **`ExtendedQueryMol::toBinary`** stores the number of tautomer queries in 16
  bits: a bundle of 65,537 round-trips as 1. *Crate:* refuses more than
  65,535.
- **`InfoBitRanker::setBiasList`** keeps the list before checking it, so a list
  it refuses still takes effect. *Crate:* checks first.
- **`DrawAttachmentLine`** passes `rawCoords` where `drawWavyLine` takes its
  vertical offset.
- **ACS 1996 drawing** sets its options from the mean bond length unchecked,
  so tiny bonds make the scale huge. *Crate:* checks the options it sets
  before drawing.

## RDKit's Python wrappers

conda-forge's Boost.Python wrappers, which the crate mirrors, have mistakes of
their own. *Crate:* follows RDKit's C++ where they differ, and documents it.

- **`CanonicalRankAtomsInFragment`** passes `includeChiralPresence` where
  `rankFragmentAtoms` takes `includeAtomMaps`, and never passes
  `includeChiralPresence` itself: by default it ranks without atom maps,
  although its docstring says `includeAtomMaps=True`.
- **Empty lists.** `pythonObjectToVect` and similar helpers test a list's
  truth, so an empty list means `None` for `AddHs`' `onlyOnAtoms`,
  `MolFragmentToSmiles`' `bondsToUse`, `FragmentOnBonds`' `bondTypes`, the
  fingerprint generators' `countBounds`, `AssignCIPLabels`' atoms and bonds,
  and `AlignPoints`' weights: an empty set of atoms or bonds is all of them.
- **`PointND.Distance`** is bound to `Point3D`'s distance function, so it
  raises for every `PointND`.
- **Misnamed and missing keywords.** `rdGaussianShape.ShapeInput` names its
  molecule `self` and gives its other three arguments no defaults, although
  the C++ constructor has them. `FilterCatalog.AddEntry` names its parameters
  `entry` and `updateFPLength`, one place off. `FeaturesValidation` spells a
  keyword `allowDummmies`.
- **Docstrings that disagree with the defaults.** The MHFP encoder's methods
  are registered with `BOOST_PYTHON_FUNCTION_OVERLOADS`, so an omitted
  `kekulize` takes the C++ function's default, true, although the docstring
  says false.
- **Duplicates.** `HasPropQueryBond` is registered three times.
- **Substructure matches.** `HasSubstructMatch` and `GetSubstructMatch` set
  `maxMatches = 1` on a copy of the parameters and then search with the
  original, so they find up to `maxMatches` (1,000 by default) to report
  one.

## RDKit's Python modules

Porting the Python modules turned up functions that fail in RDKit 2026.09.1
with conda-forge's Python 3.12 and NumPy 2, or that return something other
than what they say. *Crate:* each port computes what the function means to,
and its documentation says how Python differs.

- **`ML.InfoTheory.BitRank`** makes arrays of the abstract
  `numpy.integer`, which NumPy 2 refuses, so `FormCounts`, `CalcInfoGains`,
  and `RankBits` raise `TypeError`. `AnalyzeSparseVects` also calls a
  `GetSize` that sparse bit vectors do not have.
- **`ML.Data.Stats.StandardizeMatrix`** takes `math.sqrt` of an array of
  deviations, which raises `TypeError` for any two-dimensional matrix; only
  one-dimensional vectors work.
- **`ML.Data.Stats.PrincipalComponents`** takes rows, not columns, of
  LAPACK's matrix of eigenvectors, and picks them by each eigenvalue's place
  in LAPACK's order, so the vectors it returns are not eigenvectors.
- **`ML.Cluster.Resemblance`.** `EuclideanDistance` sizes its array with a
  float, and `CalcMetricMatrix` hands the metric a `map` object, which has no
  length. Both raise `TypeError`.
- **`ML.files.ReadDataFile`** assigns a `map` object to a NumPy row and
  raises `TypeError`.
- **`RDRandom.shuffle(x)`** without its `random` argument reads a
  `random._randbelow` that Python 3.12 does not have.
- **`Chem.SupplierFromFilename`** passes its delimiter to `TDTMolSupplier`,
  which takes none, so it fails for every `.tdt` file.
- **`Chem.FragmentCatalog`.** `ProcessGainsFile` calls a file's Python 2
  `xreadlines`, so it always fails, and `BuildAdjacencyList` with
  `orderLevels` raises `NameError`, since `argsort` is never imported.
- **`Chem.BuildFragmentCatalog`.** `ShowDetails` calls
  `FragCatalog.GetFragDescription`, which does not exist, so it raises
  `AttributeError` for any gains. `ScoreMolecules` counts bit `id` in row
  `id - 1`, so bit 0 lands in the last row (the port keeps this, since the
  script's output depends on it).
- **`Chem.ChemUtils.BulkTester`** calls `Randomize.CheckCanonicalization`,
  which RDKit removed, so it fails every molecule that sanitizes, and calls
  a supplier's Python 2 `next` method. *Crate:* not ported, with a reason.
- **`UnitTestFunctionalGroups.test4Hs`** fails: it asserts that adding
  hydrogens never changes a functional-group fingerprint, but they change
  725 of the 4,978 in its data set.
