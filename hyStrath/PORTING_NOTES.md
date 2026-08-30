# hyStrath: OpenFOAM v1706 -> v2412 port notes

This document tracks the state of the port from OpenFOAM v1706 (hyStrath's
original target) to v2412.

**Important correction (superseded an earlier draft of this file):** an
earlier pass renamed the turbulence framework to `momentumTransportModels`,
based on a mix-up between OpenFOAM's two active forks. That rename belongs
to the OpenFOAM Foundation's `.org`/`dev` branch (OpenFOAM-11+). This project
targets the ESI/OpenCFD `.com` branch, where "v2412" means
`OpenFOAM-v2412.tgz` from https://sourceforge.net/projects/openfoam/files/ —
and on that branch, v1706's names are still exactly correct:
`turbulenceModel`, `compressibleTurbulenceModels`,
`incompressibleTurbulenceModels`, `compressibleTransportModel`,
`basicThermo`, etc. All of that was reverted.

**This was then re-verified directly against the real source**, not from
memory: `OpenFOAM-v2412.tgz` was downloaded from SourceForge and diffed
file-by-file against hyStrath's vendored copies and header list. That
download and the diffs are reproducible by anyone with SourceForge access;
they are not guesses.

## What changed, and why (verified against real v2412 source)

1. **Two entire vendored duplicate libraries were removed, not ported:**
   - `hyStrath/src/TurbulenceModels/{compressible,schemes}` was a verbatim
     copy of OpenFOAM's own stock compressible turbulence library
     (`CompressibleTurbulenceModel`, `turbulentFluidThermoModel`, wall
     function BCs, `DEShybrid`). Diffing every file against the real
     `OpenFOAM-v2412/src/TurbulenceModels/{compressible,schemes}` shows only
     copyright-header churn and small C++ modernisation (e.g. a new pure
     virtual `devRhoReff(const volVectorField&)` overload on
     `compressibleTurbulenceModel`, `= delete` instead of private
     copy-ctor/assignment, and an improved `DEShybrid` with Grey Area
     Mitigation). None of it carried a hyStrath change. Note: several of the
     wall-function BCs (`temperatureCoupledBase`,
     `turbulentTemperatureCoupledBaffleMixed`, `alphatJayatillekeWallFunction`,
     etc.) moved in v2412 to a new library, `src/thermoTools`
     (`libthermoTools`) — confirmed no hyStrath tutorial case
     (`run/**`) references any of these specific BC types by name, so no
     replacement dependency was needed.
   - `hyStrath/src/fvOptions` was a verbatim copy of OpenFOAM's entire stock
     `fvOptions` library (cellSetOption, every general/derived source,
     constraint, correction, interRegion model) — the *original author's own
     comment* in `Make/files` says "ONLY MODIF MADE IS HERE" pointing at
     `variableHeatTransfer.C`. Diffing that one file against the real
     `OpenFOAM-v2412/src/fvOptions/.../variableHeatTransfer.C` shows the
     upstream version modernised a few dictionary calls
     (`lookup()`/`readScalar(lookup())` -> `readEntry()`,
     `lookupOrDefault<T>()` -> `getOrDefault<T>()`) but the type names
     hyStrath's fork actually touches — `compressible::turbulenceModel`,
     `interRegionHeatTransferModel` — are unchanged.

   Since v2412 ships both stock libraries natively under their *original*
   names, re-deriving 300+ files of someone else's already-shipped library
   is unnecessary risk. Both directories were trimmed to their one genuine
   addition, with dependents pointed at the real libraries:
   - `TurbulenceModels/turbulenceModels/`: kept `omegaLowReWallFunction`
     (a real hyStrath BC, not present in stock OpenFOAM), needs no source
     changes — confirmed it only touches the untouched
     `fixedValueFvPatchField`/`Pstream::commsTypes` API.
   - `fvOptions/`: kept `variableHeatTransfer.C`, renamed to
     `multi2VariableHeatTransfer` (class + `TypeName` string) to avoid a
     runtime-selection-table clash with OpenFOAM's own stock
     `variableHeatTransfer` fvOption, now that both libraries link together.
     Confirmed no `run/` tutorial case references the old type name.
   - Dependents (`hy2Foam`, `hy2MhdFoam`, `functionObjects/field`,
     `functionObjects/forces`, `hTCModels`, `fvOptions` itself) now link the
     real `-lcompressibleTurbulenceModels -lfvOptions` in addition to their
     own trimmed `-lstrathFvOptions`, instead of the deleted
     `-lstrathCompressibleTurbulenceModels` duplicate.

   Everything else — `transportModels/compressible`,
   `compressible::turbulenceModel`, `basicThermo`, `fluid2Thermo`'s
   `compressibleTransportModel` base — is untouched, because the real v2412
   source confirms these are unchanged from v1706.

## Broad header-surface check against the real source

Every `#include "*.H"` used anywhere in hyStrath was extracted (389 total),
hyStrath's own headers subtracted (277), leaving 162 references to stock
OpenFOAM headers. Cross-checking those 162 names against every header
filename that exists anywhere in the real `OpenFOAM-v2412.tgz` (5502 headers
total): **161 of 162 exist unchanged**. The one exception,
`turbulenceModel2.H` (included from
`src/mhdModel/submodels/conductivityModels/1/laminar/laminar2.H`), turned
out to be **pre-existing dead code**: the class `turbulenceModel2` it wants
isn't declared anywhere in hyStrath either, and the containing
`conductivityModels/1/` directory is a leftover/alternate copy that isn't
wired into `src/mhdModel`'s `Allwmake` at all (only the sibling
`conductivityModels/` without the `/1` is built). This dangling include
predates this porting session and was never compiled even under v1706 — it
is not a regression.

This is strong evidence that hyStrath's actual dependency surface on
OpenFOAM has barely moved across this 7-year span on the `.com` branch.
`ODESystem` (the base of hyStrath's own `chemistry2Model`) was spot-checked
directly and its `nEqns()`/`derivatives()`/`jacobian()` interface is
byte-for-byte the same in v2412 as it was in v1706.

## What was checked and found to be low-risk (now backed by the source diff above)

hyStrath's actual physics code (`strathReactionThermo`, `strathSpecie`,
`strathChemistryModel`, `mhdModel`, `hTCModels`) is **not** built on
OpenFOAM's `psiThermo`/`rhoThermo`/`fluidThermo` or
`BasicChemistryModel`/`StandardChemistryModel` template hierarchies —
instead it's a self-contained parallel "2-temperature" hierarchy
(`basic2Thermo`/`fluid2Thermo`/`he2Thermo`/`multi2Thermo`/`rho2Thermo`/
`rho2ReactionThermo`, `chemistry2Model`/`chemistry2Solver`/
`chemistry2Reader`) built directly on low-level primitives (`IOdictionary`,
`ODESystem`, `dictionary`, `volScalarField`, `autoPtr`) that the header- and
interface-level checks above confirm are unchanged. This ~300-file module
should need little to no source change to compile.

## Not yet done / still needs your compiler's feedback

- **Nothing here has actually been compiled** — there is no OpenFOAM install
  in the porting environment (only its source tarball, downloaded for
  read-only comparison). The checks above are real file/header diffs against
  the genuine v2412 release, not guesses, but a full build can still surface
  something a name-level/interface-level check can't (subtle signature
  changes, macro expansion differences, wmake/compiler-flag issues).
- **`hy2MhdFoam`, utilities (`makeAxialMesh`, `blockMeshDG`), and the
  remaining solver-level files** (`hy2Foam`'s `eqns/`, `BCs/`, `numerics/`,
  `runTimeEditing/`, `LTS/`) have not been individually read line-by-line
  beyond the header-surface and symbol greps described above.
- **Case dictionaries under `run/`** (`turbulenceProperties`,
  `thermophysicalProperties`, `fvSchemes`, `fvSolution`, etc.) have not been
  checked for keyword/format drift between v1706 and v2412.
- **wmake toolchain / C++ standard**: not exhaustively checked for conflicts
  with whatever standard v2412's `wmake` defaults to.
- The `hyPoliMi` suite (targeting v1912) was explicitly out of scope for this
  pass per your instructions.
- The `libthermoTools` dependency noted above only matters if you later want
  to use `temperatureCoupledBase`, `turbulentTemperatureCoupledBaffleMixed`,
  `alphatJayatillekeWallFunction`, or the other wall functions that moved
  there — none of the current tutorials need it.

## Suggested next step

Build `hyStrath` against a real OpenFOAM v2412 install
(`./install-all.sh <np> 2>&1 | tee log.install`) and send back the compiler
errors — given how little of the real API surface has moved, there should be
few, and they'll be fast to fix from real errors rather than further
speculative edits.
