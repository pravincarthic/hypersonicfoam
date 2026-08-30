# hyStrath: OpenFOAM v1706 -> v2412 port notes

This document tracks the state of the port from OpenFOAM v1706 (hyStrath's
original target) to v2412, done without access to the real v2412 source tree
or a compiler in the porting environment. Treat this as a best-effort draft:
**compile it against a real OpenFOAM v2412 install and use the errors to
find anything below that's wrong.**

## What changed, and why (high confidence)

1. **`turbulenceModels` -> `momentumTransportModels`.** OpenFOAM restructured
   its turbulence framework starting around v1906 so the same classes serve
   single-phase and multiphase momentum transport. `Foam::turbulenceModel` ->
   `Foam::momentumTransportModel`, and the `compressible::`/`incompressible::`
   namespaced typedefs followed suit. The library moved from
   `src/TurbulenceModels/turbulenceModels` to
   `src/MomentumTransportModels/momentumTransportModels`, and
   `libturbulenceModels`/`libincompressibleTurbulenceModels` became
   `libmomentumTransportModels`/`libincompressibleMomentumTransportModels`.
   All `Make/options` and all source references to these symbols in hyStrath
   were updated (see the two `Port build system...` / `Rename
   turbulenceModel...` commits).

2. **`transportModels/compressible` and `transportModels/incompressible`
   removed.** Viscosity/conductivity access moved directly onto the thermo
   classes (`mu()`, `kappa()`, etc. on `fluidThermo`) years ago. All
   `-I.../transportModels/...` includes and `-lcompressibleTransportModels`/
   `-lincompressibleTransportModels` link lines were dropped. The one place
   hyStrath's own code inherited from the now-gone `compressibleTransportModel`
   base (`fluid2Thermo`) had that base class removed — `fluid2Thermo` already
   implements its own `mu()`/`nu()`, so nothing was lost.

3. **Two entire vendored duplicate libraries were removed, not ported:**
   - `hyStrath/src/TurbulenceModels/{compressible,schemes}` was a verbatim,
     unmodified copy of OpenFOAM's own stock compressible turbulence library
     (`CompressibleTurbulenceModel`, `turbulentFluidThermoModel`, all the
     wall-function BCs, `DEShybrid`). None of it carried a hyStrath change.
   - `hyStrath/src/fvOptions` was a verbatim copy of OpenFOAM's entire stock
     `fvOptions` library (cellSetOption, every general/derived source,
     constraint, correction, interRegion model) — the *original author's own
     comment* in `Make/files` says "ONLY MODIF MADE IS HERE" pointing at
     `variableHeatTransfer.C`.

   Since v2412 ships both stock libraries natively, re-deriving 300+ files of
   someone else's already-ported library from memory (with no compiler to
   check against) is much higher risk than just linking the real thing. Both
   directories were trimmed to their one genuine addition:
   - `TurbulenceModels/turbulenceModels/`: kept `omegaLowReWallFunction`
     (a real hyStrath BC), needs no changes — the code only touches the very
     stable `fixedValueFvPatchField`/`Pstream` API.
   - `fvOptions/`: kept `variableHeatTransfer.C`, renamed to
     `multi2VariableHeatTransfer` (both the class and its `TypeName` string)
     to avoid a runtime-selection-table clash with OpenFOAM's own
     `variableHeatTransfer` fvOption, since both libraries are now linked
     together. Confirmed no run/ tutorial case references the old name.

   Dependents (`hy2Foam`, `hy2MhdFoam`, `functionObjects/field`,
   `functionObjects/forces`, `hTCModels`, `fvOptions` itself) now link the
   real `-lmomentumTransportModels -lcompressibleMomentumTransportModels
   -lfvOptions` instead of the deleted `-lstrathCompressibleTurbulenceModels`
   duplicate.

## What was checked and found to be low-risk (medium-high confidence)

hyStrath's actual physics code (`strathReactionThermo`, `strathSpecie`,
`strathChemistryModel`, `mhdModel`, `hTCModels`) is **not** built on
OpenFOAM's `psiThermo`/`rhoThermo`/`fluidThermo` or
`BasicChemistryModel`/`StandardChemistryModel` template hierarchies, which is
where most of the v1706->v2412 API churn actually happened. Instead it's a
self-contained parallel "2-temperature" hierarchy
(`basic2Thermo`/`fluid2Thermo`/`he2Thermo`/`multi2Thermo`/`rho2Thermo`/
`rho2ReactionThermo`, `chemistry2Model`/`chemistry2Solver`/`chemistry2Reader`)
built directly on very stable low-level primitives: `IOdictionary`,
`ODESystem`, `dictionary`, `volScalarField`, `autoPtr`. These interfaces have
not meaningfully changed across this span, so this ~300-file module should
need little to no source change to compile. It was not rewritten speculatively
to avoid introducing unverified guesses into otherwise-working code.

## Not yet done / needs your compiler's feedback

- **This has not been compiled.** There is no OpenFOAM install or network
  access to the real v2412 source in the porting environment. Everything
  above is applied from documented/known OpenFOAM API history, not verified
  against actual v2412 headers.
- **`hy2MhdFoam`, utilities (`makeAxialMesh`, `blockMeshDG`), and the
  remaining solver-level files** (`hy2Foam`'s `eqns/`, `BCs/`, `numerics/`,
  `runTimeEditing/`, `LTS/`) have not been individually audited beyond the
  grep sweeps described above.
- **Case dictionaries under `run/`** (`turbulenceProperties`,
  `thermophysicalProperties`, `fvSchemes`, `fvSolution`, etc.) have not been
  checked for keyword/format drift between v1706 and v2412. The dictionary
  *file name* `turbulenceProperties` is believed to still be correct in
  v2412 (via `momentumTransportModel::propertiesName`), but block contents
  were not audited.
- **wmake toolchain**: `wmake`'s required C++ standard moved from C++11 (v1706)
  to C++14/17 in later versions. No code was found using anything that would
  conflict with a newer standard, but this wasn't exhaustively checked.
- The `hyPoliMi` suite (targeting v1912) was explicitly out of scope for this
  pass per your instructions.

## Suggested next step

Build `hyStrath` against a real OpenFOAM v2412 install
(`./install-all.sh <np> 2>&1 | tee log.install`) and send back the compiler
errors — they'll pinpoint exactly which of the "low-risk" assumptions above
don't hold, and fixing from real errors will be much faster and more
reliable than further speculative edits.
