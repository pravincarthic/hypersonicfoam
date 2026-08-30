# hyStrath: OpenFOAM v1706 -> v2412 port notes

## Status: builds clean, end to end, against real installed OpenFOAM v2412

This is no longer a source-level, unverified port. OpenFOAM v2412
(2412.260127-1, from the official openfoam.com apt repository) was
installed and hyStrath was actually built against it with the
`CMakeLists.txt` in this directory:

```
source /usr/lib/openfoam/openfoam2412/etc/bashrc
cd hyStrath
rm -rf build && cmake -S . -B build -DCMAKE_BUILD_TYPE=Release
cmake --build build -j $(nproc)
```

Result: exit code 0, zero compiler errors, all 20 hyStrath libraries plus
`hy2Foam` and `hy2MhdFoam` built and linked, and `hy2Foam -help` runs and
correctly reports `Using: OpenFOAM-2412`. Everything below reflects real
compiler/linker feedback, not inference from reading headers.

wmake itself was not exercised (only the CMake build) — the two build
systems build the same sources against the same install, but if you use
`./install-all.sh` / wmake instead, treat that as a second, independent
data point and expect it to need the *same* source fixes listed below
(they're in the `.C`/`.H` files, not the build system).

## The two genuinely wrong turns this port took, and how they were caught

1. **`momentumTransportModels` rename.** An early pass renamed the
   turbulence framework based on a mix-up between OpenFOAM's two forks —
   that rename belongs to the OpenFOAM Foundation's `.org`/`dev` branch
   (OpenFOAM-11+), not the ESI/OpenCFD `.com` branch this project targets
   (where "v2412" means `OpenFOAM-v2412.tgz` from
   https://sourceforge.net/projects/openfoam/files/). Caught by downloading
   the real tarball and diffing against it; fully reverted non-destructively
   via `git revert`.
2. **`TurbulenceModels/compressible` wrongly deleted as an "unmodified
   vendored duplicate."** That conclusion was based on spot-diffing only 2
   of the directory's 51 files. A real build surfaced a
   `ThermalDiffusivity<CompressibleTurbulenceModel<fluidThermo>>::New(...)`
   signature mismatch that traced back to this: hyStrath actually retargets
   `turbulentFluidThermoModel.H`'s `compressible::turbulenceModel` typedef
   chain from stock `fluidThermo` to its own `multi2Thermo` — a genuine,
   load-bearing customization missed by the shallow diff. Restored from git
   history (all 51 files) and ported as `strathCompressibleTurbulenceModel`.
   Lesson generalized: every "this is just a vendored copy" conclusion in
   this codebase must be a full per-file diff, not a sample.

`TurbulenceModels/schemes` (`DEShybrid`) and everything in `fvOptions`
except `variableHeatTransfer.C` remain confirmed-unmodified duplicates —
those two *were* fully diffed file-by-file, not sampled, and stay removed
in favour of the real stock libraries v2412 ships.

## Real source-level fixes applied (all confirmed by the compiler)

Grouped by root cause, since most appeared at more than one call site:

- **`declareRunTimeSelectionTable`'s generated HashTable typedef renamed**
  from `<prefix>ConstructorTable` to `<prefix>ConstructorTableType`
  (`<prefix>ConstructorTable` is now a convenience static lookup function
  instead). Every hand-written `New()` selector using the classic
  `typename XConstructorTable::iterator` idiom needed
  `XConstructorTableType::iterator` — 22 files across strathSpecie,
  strathReactionThermo, strathChemistryModel, hTCModels, mhdModel, and
  blockMeshDG.
- **`dictionary::lookup()` no longer implicitly converts to a target type**
  (it returns `ITstream`, which has no such conversion). Every
  `word x = dict.lookup("key")` / `const word& x = dict.lookup(...)` /
  `vector x = dict.lookup(...)` became `dict.get<T>("key")` — Reaction2.C,
  GuptaMR.C, createEMField.H, createReactingFields.H, makeAxialMesh.C.
- **`Foam::string`/`word` lost `operator()(pos, len)` for substring
  extraction** (consolidated into `.substr(pos, len)`; `operator()` is now
  reserved for regex matching) — Reaction2.C, rho2HTCModelNew.C.
- **`word` lost its `(label)` constructor** — `word(i)` for a label `i` is
  now `Foam::name(i)` — hTC2Model.C, noHTC2.C, laminar2.C.
- **`Istream::readBegin()`/`readEnd()` changed return type from `Istream&`
  to `bool`**, breaking the `member_(readScalar(is.readBegin(name)))`
  initializer-list chaining idiom — Arrhenius2ReactionRate, Shatalov,
  HoffertLien reaction rates; fixed by calling `readBegin()` as its own
  statement and reading members via `is >> a >> b >> c` before `readEnd()`.
- **`Ostream::writeEntryIfDifferent<T>()` moved from a free function to a
  member function on `Ostream`** — `writeEntryIfDifferent<word>(os, k, a,
  b)` became `os.writeEntryIfDifferent<word>(k, a, b)` — 7 call sites
  across fixedRho and the nonEq*/nonEqMaxwellSlip wall functions.
- **`functionObjects::writeFile::writeTime()` renamed to
  `writeCurrentTime()`** — 7 call sites across specieReactionRates and
  functionObjects/field/forces.
- **`autoPtr<T>(NULL)` is ambiguous** (no longer resolves against a single
  overload) — needs `autoPtr<T>(nullptr)` — mhdModel.C,
  blockMeshDG's block/blockDescriptor/curvedEdge headers.
- **`xferCopy()` removed** (pre-C++11 transfer-semantics helper, superseded
  by move semantics) — blockMeshDG's blockMeshTopology.C/blockMeshApp.C;
  fixed with a direct pass-by-value or an explicit `pointField(...)` copy
  where the target parameter is now `pointField&&`.
- **`coordinateSystem`'s `(objectRegistry&, dictionary&)` constructor
  removed**, replaced by a family of `coordinateSystem::New(...)` static
  selectors (some private, some public — the public
  `New(modelType, obr, dict, readOrigin)` overload is the one that fits) —
  one call site in `forces.C`.
- **Two latent, always-broken dead accessors**, never actually callable
  even under v1706, that a newer GCC's more eager template instantiation
  turned into hard errors instead of silently-never-instantiated code:
  `Reaction2::name()` (non-const overload trying to bind a mutable
  reference to a `const word` member) and `rampInletFvPatchField`'s
  `amplitude()`/`timeDuration()` (declared to return `scalar`/`scalar&`
  while the real members are `autoPtr<Function1<scalar>>`, per the actual,
  working implementation in the `.C` file). Both removed rather than
  "fixed", since nothing in the codebase calls them.
- **A retired RAS model**: `v2f` (k-ε-f) no longer exists anywhere in
  v2412; removed its one `#include`/`makeRASModel` registration from
  `turbulentFluidThermoModels.C` (every other RAS/LES model registered
  there was confirmed still present).
- **A few missing include paths**, not source-level API changes: a
  transitively-required `thermophysicalModels/thermophysicalProperties`
  (stock `icoTabulated.H` now pulls in
  `nonUniformTableThermophysicalFunction.H` from there), a missing
  `multi2Thermo.H` include in three functionObjects/field files, the bare
  (non-`lnInclude`) `transportModels` parent directory that
  `turbulentTransportModel.H`'s relative-path `#include
  "incompressible/transportModel/transportModel.H"` needs, and each
  solver's own source directory plus the `CREATE_FIELDS` macro override
  that `postProcess.H` needs to find `hy2Foam_createFields.H` instead of
  a literal `createFields.H`.
- **Link-time only**: `libOpenFOAM.so` carries undefined `UPstream::`
  references that only need resolving at the final executable link (ELF
  shared libraries are allowed undefined symbols) — added the OpenFOAM-
  provided dummy (non-MPI) `libPstream` to the two solver executables only,
  matching wmake's own "link dummy stub" handling in its Gcc link rules.

## Confirmed correct as originally written (checked, not assumed)

- `transportModels/compressible`, `compressible::turbulenceModel` (the
  class itself, as opposed to the typedef file above), `basicThermo`,
  `fluid2Thermo`'s `compressibleTransportModel` base, `HashTable::iterator`
  / `.find()`/`.end()` (the selector idiom itself, once the typedef rename
  above is applied), `simpleMatrix`, `ODESystem`/`ODESolver::New`, and
  `printTable` are all unchanged from v1706 to v2412 on this branch.
- Every one of hyStrath's own physics libraries (`strathReactionThermo`,
  `strathSpecie`, `strathChemistryModel`, `mhdModel`, `hTCModels`) is
  self-contained on stable low-level primitives (`IOdictionary`,
  `ODESystem`, `dictionary`, `volScalarField`, `autoPtr`), not on the
  OpenFOAM template hierarchies that actually saw churn
  (`psiThermo`/`rhoThermo`, `BasicChemistryModel`) — this is why so much of
  the ~300-file thermophysicalModels tree needed zero changes.

## Known dead code left alone (pre-existing, confirmed unused, harmless)

- `chemistryModel.Shat/`, `reaction/Reactions.Shat/`,
  `mhdModel/submodels/conductivityModels/1/` — stale/alternate copies never
  wired into any `Make/files`, excluded from the CMake include-directory
  scan (`HYSTRATH_DEAD_DIR_PATTERNS` in `CMakeLists.txt`) specifically
  because some contain headers with the *same name* as their live
  counterpart, which caused a real, silent wrong-header bug (`make2Reactions.C`
  compiling `Reactions.Shat/Reaction/Reaction2.C`) before being excluded.
- `laminar2.H`'s `#include "turbulenceModel2.H"` (a class that doesn't
  exist anywhere in this tree) is inside the dead `conductivityModels/1/`
  copy above, so it's never actually compiled.
- `hy2Foam_solver.H` references `HY2FOAM_EXTERNAL_FILE_HYBRID_COUPLING`
  and `HY2FOAM_EXTERNAL_FILE_OUTPUT`, never `#define`d anywhere — this is
  **not** a bug, both are wrapped in `#ifdef`/`#endif` as optional
  extension points (for coupling with an external application, or custom
  per-timestep output) that compile away to nothing when left undefined.

## Still not done

- **Runtime/physics correctness.** This confirms the code *compiles and
  links* correctly, not that the two-temperature reacting-flow physics
  produces correct results. Run an actual tutorial case
  (`hyStrath/run/hy2Foam/...`) through the solver and check it against a
  known-good v1706 result before trusting it for real work.
- **Case dictionaries under `run/`** (`turbulenceProperties`,
  `thermophysicalProperties`, `fvSchemes`, `fvSolution`, etc.) have not
  been checked for keyword/format drift between v1706 and v2412.
- **`hyPoliMi`** (targeting v1912) was out of scope for this pass.
- **`makeAxialMesh`/`blockMeshDG`** build and link cleanly but haven't been
  run against real geometry.
- The `libthermoTools` wall-function BCs that moved out of
  `TurbulenceModels/compressible` in real v2412
  (`temperatureCoupledBase`, `turbulentTemperatureCoupledBaffleMixed`,
  `alphatJayatillekeWallFunction`, etc.) aren't linked by this build,
  since no current tutorial case references them by name; add
  `-I$(LIB_SRC)/thermoTools/lnInclude -lthermoTools` if a case ever needs
  one.
