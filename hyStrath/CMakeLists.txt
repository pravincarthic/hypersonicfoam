#[=======================================================================[
  hyStrath - CMake build (alternative to wmake)

  This is a from-scratch CMake translation of hyStrath's wmake Make/files
  and Make/options, written against OpenFOAM v2412 (verified directly
  against the real OpenFOAM-v2412.tgz source and its wmake/rules/*
  compiler rules - see PORTING_NOTES.md for how that verification was
  done). It builds the same libraries and the hy2Foam / hy2MhdFoam
  solvers that `./install-all.sh` would, side by side with wmake, without
  needing wmake itself.

  Usage:
      source /path/to/OpenFOAM-v2412/etc/bashrc     # required, see below
      cd hyStrath
      cmake -B build -DCMAKE_BUILD_TYPE=Release
      cmake --build build -j $(nproc)
      cmake --install build     # optional; libraries/solvers already land
                                 # directly in FOAM_USER_LIBBIN/APPBIN below

  Requirements:
    - An OpenFOAM v2412 environment must already be sourced (its own
      etc/bashrc), so that WM_PROJECT_DIR, FOAM_SRC, FOAM_LIBBIN,
      FOAM_USER_LIBBIN, FOAM_USER_APPBIN, WM_PRECISION_OPTION,
      WM_LABEL_SIZE and WM_COMPILE_OPTION are all set in the environment
      CMake is invoked from. This build reads them directly rather than
      re-deriving OpenFOAM's own install layout.
    - OpenFOAM v2412 itself must already be built (its own lnInclude
      directories and shared libraries must exist under
      $WM_PROJECT_DIR/src/*/lnInclude and $FOAM_LIBBIN) - this project
      only builds hyStrath's own libraries/solvers against that install,
      it does not build OpenFOAM itself.

  Note: hy2Foam_solver.H also references two further macros,
  HY2FOAM_EXTERNAL_FILE_HYBRID_COUPLING and HY2FOAM_EXTERNAL_FILE_OUTPUT,
  which are never #define'd anywhere in the source tree - this is not a
  bug: both uses are wrapped in #ifdef/#endif, so they are optional
  extension points (e.g. for coupling hy2Foam with an external
  application, or adding custom per-timestep output) that compile away to
  nothing when left undefined, exactly as intended.
#]=======================================================================]

cmake_minimum_required(VERSION 3.16)
project(hyStrath CXX)

# ============================================================================
# 1. Locate the OpenFOAM v2412 environment
# ============================================================================

if(NOT DEFINED ENV{WM_PROJECT_DIR})
  message(FATAL_ERROR
    "OpenFOAM environment is not sourced in this shell.\n"
    "Run e.g.\n"
    "    source /path/to/OpenFOAM-v2412/etc/bashrc\n"
    "and re-run cmake from a fresh build directory.")
endif()

set(FOAM_SRC          "$ENV{FOAM_SRC}")
set(FOAM_LIBBIN       "$ENV{FOAM_LIBBIN}")
set(FOAM_USER_LIBBIN  "$ENV{FOAM_USER_LIBBIN}")
set(FOAM_USER_APPBIN  "$ENV{FOAM_USER_APPBIN}")

if(FOAM_SRC STREQUAL "" OR FOAM_USER_LIBBIN STREQUAL "" OR FOAM_USER_APPBIN STREQUAL "")
  message(FATAL_ERROR
    "FOAM_SRC / FOAM_USER_LIBBIN / FOAM_USER_APPBIN are not set.\n"
    "These come from OpenFOAM's own etc/bashrc (they also require "
    "WM_PROJECT_USER_DIR - i.e. $HOME/OpenFOAM/$USER-v2412 - to resolve). "
    "Source OpenFOAM's environment before configuring.")
endif()

file(MAKE_DIRECTORY "${FOAM_USER_LIBBIN}")
file(MAKE_DIRECTORY "${FOAM_USER_APPBIN}")

message(STATUS "hyStrath / OpenFOAM v2412 CMake build")
message(STATUS "  WM_PROJECT_VERSION : $ENV{WM_PROJECT_VERSION}")
message(STATUS "  FOAM_SRC           : ${FOAM_SRC}")
message(STATUS "  FOAM_LIBBIN        : ${FOAM_LIBBIN}")
message(STATUS "  FOAM_USER_LIBBIN   : ${FOAM_USER_LIBBIN}")
message(STATUS "  FOAM_USER_APPBIN   : ${FOAM_USER_APPBIN}")

# ============================================================================
# 2. Compiler setup
#    Mirrors OpenFOAM v2412's own wmake/rules/linux64Gcc/c++ and
#    wmake/rules/General/general, verified against the real release
#    tarball. If your install uses a non-default WM_LABEL_SIZE /
#    WM_PRECISION_OPTION (check with `echo $WM_LABEL_SIZE
#    $WM_PRECISION_OPTION` after sourcing OpenFOAM's environment), this
#    picks them up automatically; the fallbacks below (32 / DP) are only
#    used if those variables are somehow unset.
# ============================================================================

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)               # wmake uses -std=c++17, not gnu++17

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release)
endif()

string(REGEX REPLACE "^v" "" _wm_version_num "$ENV{WM_PROJECT_VERSION}")
if(_wm_version_num STREQUAL "")
  set(_wm_version_num 2412)
endif()

set(_wm_precision "$ENV{WM_PRECISION_OPTION}")
if(_wm_precision STREQUAL "")
  set(_wm_precision "DP")
endif()

set(_wm_label_size "$ENV{WM_LABEL_SIZE}")
if(_wm_label_size STREQUAL "")
  set(_wm_label_size "32")
endif()

add_compile_definitions(
  OPENFOAM=${_wm_version_num}
  WM_${_wm_precision}
  WM_LABEL_SIZE=${_wm_label_size}
  NoRepository
)

add_compile_options(
  -m64 -pthread -fPIC
  -ftemplate-depth-100
  -Wall -Wextra -Wold-style-cast -Wnon-virtual-dtor
  -Wno-unused-parameter -Wno-invalid-offsetof -Wno-attributes -Wno-unknown-pragmas
  $<$<CONFIG:Release>:-O2>
  $<$<CONFIG:RelWithDebInfo>:-O2>
  $<$<CONFIG:RelWithDebInfo>:-g>
  $<$<CONFIG:Debug>:-O0>
  $<$<CONFIG:Debug>:-g>
)

# CMake's own per-build-type defaults (e.g. Release => -O3 -DNDEBUG) would
# stack on top of the flags above and diverge from wmake's actual -O2 (no
# -DNDEBUG) Opt build; blank them out so add_compile_options() above is the
# only source of optimisation/debug flags.
foreach(_cfg RELEASE RELWITHDEBINFO DEBUG MINSIZEREL)
  set(CMAKE_CXX_FLAGS_${_cfg} "")
endforeach()

link_directories("${FOAM_LIBBIN}" "${FOAM_USER_LIBBIN}")

# wmake injects these into every single compile/link via PROJECT_INC /
# PROJECT_LIBS in wmake/makefiles/general - not via any module's own
# Make/options - since every OpenFOAM library and application needs
# OpenFOAM's own core headers (word.H, List.H, scalar.H, ...) and
# libOpenFOAM.so itself. Verified directly against that file in the real
# v2412 release rather than assumed.
include_directories(
  "${FOAM_SRC}/OpenFOAM/lnInclude"
  "${FOAM_SRC}/OSspecific/POSIX/lnInclude"
)
link_libraries(OpenFOAM dl)

set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${FOAM_USER_LIBBIN}")
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${FOAM_USER_APPBIN}")

# Match wmake: built libraries/solvers find OpenFOAM's own .so's at runtime
# without needing LD_LIBRARY_PATH tricks beyond what sourcing bashrc already sets.
set(CMAKE_BUILD_WITH_INSTALL_RPATH ON)
set(CMAKE_INSTALL_RPATH "${FOAM_LIBBIN};${FOAM_USER_LIBBIN}")
set(CMAKE_SKIP_BUILD_RPATH OFF)

# ============================================================================
# 3. Helpers
# ============================================================================

set(HY "${CMAKE_CURRENT_SOURCE_DIR}")

# hyStrath's own modules don't have wmake's per-library lnInclude symlink
# farm (that's generated by wmake itself at build time). Rather than
# generating one, every subdirectory that contains a header in a given
# module is added directly as an include directory - this reproduces the
# same "any header, anywhere in the module, is reachable by bare name"
# effect that lnInclude gives wmake builds.
#
# A handful of directories in this tree are stale/alternate copies that
# were never wired into any Make/files (e.g. chemistryModel.Shat/,
# reaction/Reactions.Shat/, mhdModel/submodels/conductivityModels/1/ - see
# PORTING_NOTES.md). They must be excluded here: several of them define
# headers with the exact same name as their live counterpart (e.g. both
# reaction/Reactions/Reaction/Reaction2.H and
# reaction/Reactions.Shat/Reaction/Reaction2.H exist), so if both ended up
# on the include path which one wins would depend on directory-glob order
# rather than which one is actually built - silently compiling the wrong,
# stale header.
set(HYSTRATH_DEAD_DIR_PATTERNS
  "\\.Shat($|/)"
  "/conductivityModels/1($|/)"
)
function(hystrath_own_include_dirs root outvar)
  file(GLOB_RECURSE _headers "${root}/*.H")
  set(_dirs "")
  foreach(_h ${_headers})
    set(_dead FALSE)
    foreach(_pat ${HYSTRATH_DEAD_DIR_PATTERNS})
      if(_h MATCHES "${_pat}")
        set(_dead TRUE)
      endif()
    endforeach()
    if(NOT _dead)
      get_filename_component(_d "${_h}" DIRECTORY)
      list(APPEND _dirs "${_d}")
    endif()
  endforeach()
  if(_dirs)
    list(REMOVE_DUPLICATES _dirs)
  endif()
  set(${outvar} "${_dirs}" PARENT_SCOPE)
endfunction()

# Convenience: turn a list of OpenFOAM library subpaths under $FOAM_SRC into
# their (already wmake-built, real) lnInclude directories.
function(hystrath_of_include_dirs outvar)
  set(_dirs "")
  foreach(_p ${ARGN})
    list(APPEND _dirs "${FOAM_SRC}/${_p}/lnInclude")
  endforeach()
  set(${outvar} "${_dirs}" PARENT_SCOPE)
endfunction()

# ============================================================================
# 4. src/thermophysicalModels/strath/strathSpecie  ->  libstrathSpecie
# ============================================================================

hystrath_own_include_dirs("${HY}/src/thermophysicalModels/strath/strathSpecie" strathSpecie_OWN_INC)
hystrath_of_include_dirs(strathSpecie_OF_INC thermophysicalModels/specie)

add_library(strathSpecie SHARED
  ${HY}/src/thermophysicalModels/strath/strathSpecie/specie/advancedSpecie/advancedSpecie.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/reaction/reactions/make2Reactions.C
)
target_include_directories(strathSpecie PUBLIC ${strathSpecie_OWN_INC} ${strathSpecie_OF_INC})
target_link_libraries(strathSpecie PUBLIC specie)

# ============================================================================
# 5. src/thermophysicalModels/strath/strathReactionThermo
#    -> libstrathReactionThermophysicalModels
# ============================================================================

hystrath_own_include_dirs("${HY}/src/thermophysicalModels/strath/strathReactionThermo" strathReactionThermo_OWN_INC)
hystrath_of_include_dirs(strathReactionThermo_OF_INC
  finiteVolume transportModels/compressible
  thermophysicalModels/basic thermophysicalModels/reactionThermo
  thermophysicalModels/specie thermophysicalModels/solidSpecie
  meshTools
)

add_library(strathReactionThermophysicalModels SHARED
  ${HY}/src/thermophysicalModels/strath/strathReactionThermo/basic/basic2Thermo/basic2Thermo.C
  ${HY}/src/thermophysicalModels/strath/strathReactionThermo/basic/fluid2Thermo/fluid2Thermo.C
  ${HY}/src/thermophysicalModels/strath/strathReactionThermo/basic/multi2Thermo/multi2Thermo.C
  ${HY}/src/thermophysicalModels/strath/strathReactionThermo/basic/rho2Thermo/rho2Thermo.C
  ${HY}/src/thermophysicalModels/strath/strathReactionThermo/basic/rho2Thermo/rho2Thermos.C
  ${HY}/src/thermophysicalModels/strath/strathReactionThermo/basic/derivedFvPatchFields/fixedRho/fixedRhoFvPatchScalarField.C
  ${HY}/src/thermophysicalModels/strath/strathReactionThermo/basic/derivedFvPatchFields/fixedTREnergy/fixed2TREnergyFvPatchScalarField.C
  ${HY}/src/thermophysicalModels/strath/strathReactionThermo/basic/derivedFvPatchFields/gradientTREnergy/gradient2TREnergyFvPatchScalarField.C
  ${HY}/src/thermophysicalModels/strath/strathReactionThermo/basic/derivedFvPatchFields/mixedTREnergy/mixed2TREnergyFvPatchScalarField.C
  ${HY}/src/thermophysicalModels/strath/strathReactionThermo/basic/derivedFvPatchFields/fixedVELMixEnergy/fixed2VELMixEnergyFvPatchScalarField.C
  ${HY}/src/thermophysicalModels/strath/strathReactionThermo/basic/derivedFvPatchFields/gradientVELMixEnergy/gradient2VELMixEnergyFvPatchScalarField.C
  ${HY}/src/thermophysicalModels/strath/strathReactionThermo/basic/derivedFvPatchFields/mixedVELMixEnergy/mixed2VELMixEnergyFvPatchScalarField.C
  ${HY}/src/thermophysicalModels/strath/strathReactionThermo/reactionThermo/chemistryReaders/chemistry2Reader/make2ChemistryReaders.C
  ${HY}/src/thermophysicalModels/strath/strathReactionThermo/reactionThermo/mixtures/basic2MultiComponentMixture/basic2MultiComponentMixture.C
  ${HY}/src/thermophysicalModels/strath/strathReactionThermo/reactionThermo/rho2ReactionThermo/rho2ReactionThermo.C
  ${HY}/src/thermophysicalModels/strath/strathReactionThermo/reactionThermo/rho2ReactionThermo/rho2ReactionThermos.C
  ${HY}/src/thermophysicalModels/strath/strathReactionThermo/reactionThermo/derivedFvPatchFields/fixedVELEnergy/fixed2VELEnergyFvPatchScalarField.C
  ${HY}/src/thermophysicalModels/strath/strathReactionThermo/reactionThermo/derivedFvPatchFields/gradientVELEnergy/gradient2VELEnergyFvPatchScalarField.C
  ${HY}/src/thermophysicalModels/strath/strathReactionThermo/reactionThermo/derivedFvPatchFields/mixedVELEnergy/mixed2VELEnergyFvPatchScalarField.C
  ${HY}/src/thermophysicalModels/strath/strathReactionThermo/reactionThermo/derivedFvPatchFields/fixedEnergy/fixed2EnergyFvPatchScalarField.C
  ${HY}/src/thermophysicalModels/strath/strathReactionThermo/reactionThermo/derivedFvPatchFields/gradientEnergy/gradient2EnergyFvPatchScalarField.C
  ${HY}/src/thermophysicalModels/strath/strathReactionThermo/reactionThermo/derivedFvPatchFields/mixedEnergy/mixed2EnergyFvPatchScalarField.C
)
target_include_directories(strathReactionThermophysicalModels PUBLIC ${strathReactionThermo_OWN_INC} ${strathReactionThermo_OF_INC})
target_link_libraries(strathReactionThermophysicalModels PUBLIC
  strathSpecie
  finiteVolume compressibleTransportModels fluidThermophysicalModels
  reactionThermophysicalModels specie solidSpecie meshTools
)

# ============================================================================
# 6. src/thermophysicalModels/strath/strathChemistryModel
#    -> libstrathChemistryModel
# ============================================================================

hystrath_own_include_dirs("${HY}/src/thermophysicalModels/strath/strathChemistryModel" strathChemistryModel_OWN_INC)
hystrath_of_include_dirs(strathChemistryModel_OF_INC
  finiteVolume transportModels/compressible
  thermophysicalModels/specie thermophysicalModels/basic
  thermophysicalModels/reactionThermo thermophysicalModels/chemistrySolver
  thermophysicalModels/functions/Polynomial thermophysicalModels/thermophysicalProperties
  ODE
)

add_library(strathChemistryModel SHARED
  ${HY}/src/thermophysicalModels/strath/strathChemistryModel/chemistryModel/basic2ChemistryModel/basic2ChemistryModel.C
  ${HY}/src/thermophysicalModels/strath/strathChemistryModel/chemistryModel/rho2ChemistryModel/rho2ChemistryModel.C
  ${HY}/src/thermophysicalModels/strath/strathChemistryModel/chemistryModel/rho2ChemistryModel/rho2ChemistryModels.C
  ${HY}/src/thermophysicalModels/strath/strathChemistryModel/chemistrySolver/chemistry2Solver/make2ChemistrySolvers.C
  ${HY}/src/thermophysicalModels/strath/strathChemistryModel/functionObjects/specieReactionRates/specieReactionRates.C
)
target_include_directories(strathChemistryModel PUBLIC ${strathChemistryModel_OWN_INC} ${strathChemistryModel_OF_INC})
target_link_libraries(strathChemistryModel PUBLIC
  strathSpecie strathReactionThermophysicalModels
  compressibleTransportModels thermophysicalProperties ODE
)

# ============================================================================
# 7. src/thermophysicalModels/strath/strathSpecie/relaxationTimes/*
# ============================================================================

foreach(_relax VTRelaxationModels VVRelaxationModels eVRelaxationModels)
  hystrath_own_include_dirs("${HY}/src/thermophysicalModels/strath/strathSpecie/relaxationTimes/${_relax}" ${_relax}_OWN_INC)
endforeach()
hystrath_of_include_dirs(relax_leaf_OF_INC finiteVolume thermophysicalModels/specie meshTools)

add_library(VTRelaxationModels SHARED
  ${HY}/src/thermophysicalModels/strath/strathSpecie/relaxationTimes/VTRelaxationModels/VTRelaxationModel/VTRelaxationModel.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/relaxationTimes/VTRelaxationModels/VTModel/VTModel.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/relaxationTimes/VTRelaxationModels/MillikanWhite/MillikanWhite.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/relaxationTimes/VTRelaxationModels/MillikanWhitePark/MillikanWhitePark.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/relaxationTimes/VTRelaxationModels/strathVT/strathVT.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/relaxationTimes/VTRelaxationModels/strathMWP/strathMWP.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/relaxationTimes/VTRelaxationModels/LeMANSMWP/LeMANSMWP.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/relaxationTimes/VTRelaxationModels/SchwartzSlawskyHerzfeld/SchwartzSlawskyHerzfeld.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/relaxationTimes/VTRelaxationModels/QCTMillikanWhitePark/QCTMillikanWhitePark.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/relaxationTimes/VTRelaxationModels/ShatalovMWP/ShatalovMWP.C
)
target_include_directories(VTRelaxationModels PUBLIC ${VTRelaxationModels_OWN_INC} ${relax_leaf_OF_INC})
target_link_libraries(VTRelaxationModels PUBLIC strathSpecie meshTools)

add_library(eVRelaxationModels SHARED
  ${HY}/src/thermophysicalModels/strath/strathSpecie/relaxationTimes/eVRelaxationModels/eVRelaxationModel/eVRelaxationModel.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/relaxationTimes/eVRelaxationModels/eVModel/eVModel.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/relaxationTimes/eVRelaxationModels/BourdonVervisch/BourdonVervisch.C
)
target_include_directories(eVRelaxationModels PUBLIC ${eVRelaxationModels_OWN_INC} ${relax_leaf_OF_INC})
target_link_libraries(eVRelaxationModels PUBLIC strathSpecie meshTools)

add_library(VVRelaxationModels SHARED
  ${HY}/src/thermophysicalModels/strath/strathSpecie/relaxationTimes/VVRelaxationModels/VVRelaxationModel/VVRelaxationModel.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/relaxationTimes/VVRelaxationModels/VVModel/VVModel.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/relaxationTimes/VVRelaxationModels/Knab/Knab.C
)
target_include_directories(VVRelaxationModels PUBLIC ${VVRelaxationModels_OWN_INC} ${relax_leaf_OF_INC})
target_link_libraries(VVRelaxationModels PUBLIC strathSpecie meshTools)

# --- relaxationTimeModels (aggregates the three above) ---
hystrath_own_include_dirs("${HY}/src/thermophysicalModels/strath/strathSpecie/relaxationTimes/relaxationTimeModels" relaxationTimeModels_OWN_INC)
hystrath_of_include_dirs(relaxationTimeModels_OF_INC
  finiteVolume meshTools transportModels/compressible
  thermophysicalModels/specie thermophysicalModels/basic
  thermophysicalModels/reactionThermo thermophysicalModels/chemistryModel
  TurbulenceModels/turbulenceModels TurbulenceModels/compressible
)

add_library(relaxationTimeModels SHARED
  ${HY}/src/thermophysicalModels/strath/strathSpecie/relaxationTimes/relaxationTimeModels/relaxationTimeModelVT/relaxationTimeModel.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/relaxationTimes/relaxationTimeModels/relaxationTimeModelVT/newRelaxationTimeModel.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/relaxationTimes/relaxationTimeModels/relaxationTimeModelVT/relaxationTimeModels.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/relaxationTimes/relaxationTimeModels/relaxationTimeModelVV/relaxationTimeModelVV.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/relaxationTimes/relaxationTimeModels/relaxationTimeModelVV/newRelaxationTimeModelVV.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/relaxationTimes/relaxationTimeModels/relaxationTimeModelVV/relaxationTimeModelsVV.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/relaxationTimes/relaxationTimeModels/relaxationTimeModelHE/relaxationTimeModelHE.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/relaxationTimes/relaxationTimeModels/relaxationTimeModelHE/newRelaxationTimeModelHE.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/relaxationTimes/relaxationTimeModels/relaxationTimeModelHE/relaxationTimeModelsHE.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/relaxationTimes/relaxationTimeModels/relaxationTimeModeleV/relaxationTimeModeleV.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/relaxationTimes/relaxationTimeModels/relaxationTimeModeleV/newRelaxationTimeModeleV.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/relaxationTimes/relaxationTimeModels/relaxationTimeModeleV/relaxationTimeModeleVs.C
)
target_include_directories(relaxationTimeModels PUBLIC ${relaxationTimeModels_OWN_INC} ${relaxationTimeModels_OF_INC})
target_link_libraries(relaxationTimeModels PUBLIC
  strathSpecie strathReactionThermophysicalModels strathChemistryModel
  VTRelaxationModels VVRelaxationModels eVRelaxationModels
  compressibleTransportModels compressibleTurbulenceModels
)

# ============================================================================
# 8. src/thermophysicalModels/strath/strathSpecie/rarefied/*
# ============================================================================

hystrath_own_include_dirs("${HY}/src/thermophysicalModels/strath/strathSpecie/rarefied/mfpModels" mfpModels_OWN_INC)

add_library(mfpModels SHARED
  ${HY}/src/thermophysicalModels/strath/strathSpecie/rarefied/mfpModels/basicMfpModel/basicMfpModel.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/rarefied/mfpModels/mfpModel/mfpModel.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/rarefied/mfpModels/maxwellian/maxwellianmfp.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/rarefied/mfpModels/hardSphere/hardSphere.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/rarefied/mfpModels/variableHardSphere/variableHardSphere.C
)
target_include_directories(mfpModels PUBLIC ${mfpModels_OWN_INC} ${relax_leaf_OF_INC})
target_link_libraries(mfpModels PUBLIC strathSpecie meshTools)

hystrath_own_include_dirs("${HY}/src/thermophysicalModels/strath/strathSpecie/rarefied/rarefactionParameters" rarefactionParameters_OWN_INC)
hystrath_of_include_dirs(rarefactionParameters_OF_INC
  finiteVolume meshTools transportModels/compressible
  thermophysicalModels/specie thermophysicalModels/basic
  thermophysicalModels/reactionThermo thermophysicalModels/chemistryModel
  TurbulenceModels/turbulenceModels TurbulenceModels/compressible
)

add_library(rarefactionParameters SHARED
  ${HY}/src/thermophysicalModels/strath/strathSpecie/rarefied/rarefactionParameters/rarefactionParameter/rarefactionParameter.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/rarefied/rarefactionParameters/rarefactionParameter/newRarefactionParameter.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/rarefied/rarefactionParameters/rarefactionParameter/rarefactionParameters.C
)
target_include_directories(rarefactionParameters PUBLIC ${rarefactionParameters_OWN_INC} ${rarefactionParameters_OF_INC})
target_link_libraries(rarefactionParameters PUBLIC
  strathSpecie strathReactionThermophysicalModels strathChemistryModel mfpModels
  compressibleTransportModels
)

# ============================================================================
# 9. src/thermophysicalModels/strath/strathSpecie/transport/*
# ============================================================================

hystrath_own_include_dirs("${HY}/src/thermophysicalModels/strath/strathSpecie/transport/mixingRules" mixingRules_OWN_INC)

add_library(mixingRules SHARED
  ${HY}/src/thermophysicalModels/strath/strathSpecie/transport/mixingRules/mixingRule/mixingRule.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/transport/mixingRules/mixingRule/newMixingRule.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/transport/mixingRules/mixingRule/mixingRules.C
)
target_include_directories(mixingRules PUBLIC ${mixingRules_OWN_INC} ${rarefactionParameters_OF_INC})
target_link_libraries(mixingRules PUBLIC
  strathSpecie strathReactionThermophysicalModels strathChemistryModel
  compressibleTransportModels
)

hystrath_own_include_dirs("${HY}/src/thermophysicalModels/strath/strathSpecie/transport/speciesDiffusion/diffusivityModels" diffusivityModels_OWN_INC)

add_library(diffusivityModels SHARED
  ${HY}/src/thermophysicalModels/strath/strathSpecie/transport/speciesDiffusion/diffusivityModels/binaryDiffusivityModel/binaryDiffusivityModel.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/transport/speciesDiffusion/diffusivityModels/noBinaryDiffusivityModel/noBinaryDiffusivityModel.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/transport/speciesDiffusion/diffusivityModels/constant/constant.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/transport/speciesDiffusion/diffusivityModels/Gupta/GuptaO.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/transport/speciesDiffusion/diffusivityModels/Gupta/GuptaD.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/transport/speciesDiffusion/diffusivityModels/Stephani/Stephani.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/transport/speciesDiffusion/diffusivityModels/Wright/WrightO.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/transport/speciesDiffusion/diffusivityModels/diffusivityModel/diffusivityModel.C
)
target_include_directories(diffusivityModels PUBLIC ${diffusivityModels_OWN_INC} ${relax_leaf_OF_INC})
target_link_libraries(diffusivityModels PUBLIC strathSpecie meshTools)

hystrath_own_include_dirs("${HY}/src/thermophysicalModels/strath/strathSpecie/transport/speciesDiffusion/multiSpeciesTransportModels" multiSpeciesTransportModels_OWN_INC)

add_library(multiSpeciesTransportModels SHARED
  ${HY}/src/thermophysicalModels/strath/strathSpecie/transport/speciesDiffusion/multiSpeciesTransportModels/multiSpeciesTransportModel/multiSpeciesTransportModel.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/transport/speciesDiffusion/multiSpeciesTransportModels/multiSpeciesTransportModel/newMultiSpeciesTransportModel.C
  ${HY}/src/thermophysicalModels/strath/strathSpecie/transport/speciesDiffusion/multiSpeciesTransportModels/multiSpeciesTransportModel/multiSpeciesTransportModels.C
)
target_include_directories(multiSpeciesTransportModels PUBLIC ${multiSpeciesTransportModels_OWN_INC} ${rarefactionParameters_OF_INC})
target_link_libraries(multiSpeciesTransportModels PUBLIC
  strathSpecie strathReactionThermophysicalModels strathChemistryModel diffusivityModels
  compressibleTransportModels
)

# ============================================================================
# 10. src/finiteVolume  ->  libstrathFiniteVolume
# ============================================================================

hystrath_own_include_dirs("${HY}/src/finiteVolume" strathFiniteVolume_OWN_INC)
hystrath_of_include_dirs(strathFiniteVolume_OF_INC finiteVolume)

add_library(strathFiniteVolume SHARED
  ${HY}/src/finiteVolume/fields/fvPatchFields/derived/rampInlet/rampInletFvPatchFields.C
  ${HY}/src/finiteVolume/fields/fvPatchFields/derived/nonCatalytic/nonCatalyticFvPatchFields.C
)
target_include_directories(strathFiniteVolume PUBLIC ${strathFiniteVolume_OWN_INC} ${strathFiniteVolume_OF_INC})
target_link_libraries(strathFiniteVolume PUBLIC OpenFOAM)

# ============================================================================
# 11. src/TurbulenceModels/turbulenceModels  ->  libstrathTurbulenceModels
# ============================================================================

hystrath_own_include_dirs("${HY}/src/TurbulenceModels/turbulenceModels" strathTurbulenceModels_OWN_INC)
hystrath_of_include_dirs(strathTurbulenceModels_OF_INC finiteVolume meshTools TurbulenceModels/turbulenceModels)

add_library(strathTurbulenceModels SHARED
  ${HY}/src/TurbulenceModels/turbulenceModels/derivedFvPatchFields/wallFunctions/omegaWallFunctions/omegaLowReWallFunction/omegaLowReWallFunctionFvPatchScalarField.C
)
target_include_directories(strathTurbulenceModels PUBLIC ${strathTurbulenceModels_OWN_INC} ${strathTurbulenceModels_OF_INC})
target_link_libraries(strathTurbulenceModels PUBLIC finiteVolume meshTools)

# ============================================================================
# 12. src/fvOptions  ->  libstrathFvOptions
# ============================================================================

hystrath_own_include_dirs("${HY}/src/fvOptions" strathFvOptions_OWN_INC)
hystrath_of_include_dirs(strathFvOptions_OF_INC
  finiteVolume meshTools sampling fvOptions
  thermophysicalModels/solidThermo thermophysicalModels/specie thermophysicalModels/basic
  transportModels transportModels/compressible transportModels/incompressible
  TurbulenceModels/turbulenceModels TurbulenceModels/compressible TurbulenceModels/incompressible
)

add_library(strathFvOptions SHARED
  ${HY}/src/fvOptions/sources/interRegion/interRegionHeatTransfer/variableHeatTransfer/multi2VariableHeatTransfer.C
)
target_include_directories(strathFvOptions PUBLIC
  ${strathFvOptions_OWN_INC} ${strathFvOptions_OF_INC}
  ${HY}/src/thermophysicalModels/strath/strathSpecie
  ${HY}/src/thermophysicalModels/strath/strathReactionThermo
)
target_link_libraries(strathFvOptions PUBLIC
  strathSpecie strathReactionThermophysicalModels
  finiteVolume sampling meshTools fvOptions
  turbulenceModels incompressibleTurbulenceModels compressibleTurbulenceModels
)

# ============================================================================
# 13. src/hTCModels  ->  libhTCModels
# ============================================================================

hystrath_own_include_dirs("${HY}/src/hTCModels" hTCModels_OWN_INC)
hystrath_of_include_dirs(hTCModels_OF_INC
  finiteVolume meshTools transportModels/compressible
  thermophysicalModels/specie thermophysicalModels/basic
  TurbulenceModels/turbulenceModels TurbulenceModels/compressible
)

add_library(hTCModels SHARED
  ${HY}/src/hTCModels/hTC2Model/hTC2Model.C
  ${HY}/src/hTCModels/rho2HTCModel/rho2HTCModel/rho2HTCModel.C
  ${HY}/src/hTCModels/rho2HTCModel/rho2HTCModel/rho2HTCModelNew.C
  ${HY}/src/hTCModels/rho2HTCModel/rho2ThermoHTC/rho2ThermoHTC.C
  ${HY}/src/hTCModels/rho2HTCModel/rho2ChemistryHTC/rho2ChemistryHTC.C
  ${HY}/src/hTCModels/noHTC/noHTC2s.C
  ${HY}/src/hTCModels/laminar/laminar2s.C
)
target_include_directories(hTCModels PUBLIC ${hTCModels_OWN_INC} ${hTCModels_OF_INC})
target_link_libraries(hTCModels PUBLIC
  strathSpecie strathReactionThermophysicalModels strathChemistryModel
  finiteVolume compressibleTransportModels compressibleTurbulenceModels
)

# ============================================================================
# 14. src/mhdModel  ->  libconductivityModels + libspbpuMhdModels
# ============================================================================

hystrath_own_include_dirs("${HY}/src/mhdModel" mhdModel_OWN_INC)

# 14a. standalone conductivityModels library (built by wmake as its own
#      target; nothing in this tree links against it, but it's kept for
#      parity with the original build).
add_library(conductivityModels SHARED
  ${HY}/src/mhdModel/submodels/conductivityModels/basicConductivityModel/basicConductivityModel.C
  ${HY}/src/mhdModel/submodels/conductivityModels/conductivityModel/conductivityModel.C
  ${HY}/src/mhdModel/submodels/conductivityModels/constSigma/constSigma.C
)
hystrath_of_include_dirs(conductivityModels_OF_INC
  finiteVolume meshTools thermophysicalModels/basic
  thermophysicalModels/specie thermophysicalModels/solidThermo thermophysicalModels/solidSpecie
)
target_include_directories(conductivityModels PUBLIC ${mhdModel_OWN_INC} ${conductivityModels_OF_INC})
target_link_libraries(conductivityModels PUBLIC
  strathReactionThermophysicalModels strathSpecie
  solidThermo finiteVolume meshTools solidSpecie radiationModels
)

# 14b. main mhdModel library. NOTE: this compiles its own copy of
#      conductivityModel.C / constSigma.C (plus the other conductivity
#      submodels) directly into libspbpuMhdModels - it does not link
#      against libconductivityModels above. This exactly matches the
#      original wmake Make/files, which lists these same source files
#      twice (once per library) rather than one linking to the other.
hystrath_of_include_dirs(spbpuMhdModels_OF_INC
  finiteVolume meshTools transportModels/compressible
  thermophysicalModels/basic thermophysicalModels/specie thermophysicalModels/solidThermo
  thermophysicalModels/SLGThermo thermophysicalModels/reactionThermo
)

add_library(spbpuMhdModels SHARED
  ${HY}/src/mhdModel/mhdModel/mhdModel.C
  ${HY}/src/mhdModel/mhdModel/mhdModelNew.C
  ${HY}/src/mhdModel/noMHD/noMHD.C
  ${HY}/src/mhdModel/lowReMag/lowReMag.C
  ${HY}/src/mhdModel/submodels/conductivityModels/conductivityModel/conductivityModel.C
  ${HY}/src/mhdModel/submodels/conductivityModels/constSigma/constSigma.C
  ${HY}/src/mhdModel/submodels/conductivityModels/Bush/Bush.C
  ${HY}/src/mhdModel/submodels/conductivityModels/Raizer/Raizer.C
  ${HY}/src/mhdModel/submodels/conductivityModels/ChapmanCowling/ChapmanCowling.C
  ${HY}/src/mhdModel/submodels/conductivityModels/SpitzerHarm/SpitzerHarm.C
  ${HY}/src/mhdModel/submodels/conductivityModels/Boltzmann/Boltzmann.C
)
target_include_directories(spbpuMhdModels PUBLIC ${mhdModel_OWN_INC} ${spbpuMhdModels_OF_INC})
target_link_libraries(spbpuMhdModels PUBLIC
  strathReactionThermophysicalModels
  finiteVolume compressibleTransportModels fluidThermophysicalModels
  specie solidThermo meshTools SLGThermo reactionThermophysicalModels
)

# ============================================================================
# 15. src/functionObjects/*
# ============================================================================

hystrath_own_include_dirs("${HY}/src/functionObjects/field" strathField_OWN_INC)
hystrath_of_include_dirs(strathField_OF_INC
  finiteVolume meshTools lagrangian/basic lagrangian/distributionModels
  fileFormats sampling surfMesh thermophysicalModels/basic
  transportModels/compressible
  TurbulenceModels/turbulenceModels TurbulenceModels/incompressible TurbulenceModels/compressible
  fvAgglomerationMethods/pairPatchAgglomeration
)

add_library(strathFieldFunctionObjects SHARED
  ${HY}/src/functionObjects/field/frictionCoefficient/frictionCoefficient.C
  ${HY}/src/functionObjects/field/pressureCoefficient/pressureCoefficient.C
  ${HY}/src/functionObjects/field/StantonNo/StantonNo.C
  ${HY}/src/functionObjects/field/yPlus/yPlus.C
  ${HY}/src/functionObjects/field/wallShearStress/wallShearStress.C
  ${HY}/src/functionObjects/field/fieldExpression/fieldExpression.C
  ${HY}/src/functionObjects/field/grad/grad.C
)
target_include_directories(strathFieldFunctionObjects PUBLIC
  ${strathField_OWN_INC} ${strathField_OF_INC}
  ${HY}/src/thermophysicalModels/strath/strathChemistryModel
  ${HY}/src/thermophysicalModels/strath/strathReactionThermo
  ${HY}/src/thermophysicalModels/strath/strathSpecie
)
target_link_libraries(strathFieldFunctionObjects PUBLIC
  strathReactionThermophysicalModels strathChemistryModel
  finiteVolume meshTools lagrangian distributionModels sampling surfMesh
  incompressibleTransportModels turbulenceModels compressibleTransportModels
  incompressibleTurbulenceModels compressibleTurbulenceModels
  pairPatchAgglomeration
)

hystrath_own_include_dirs("${HY}/src/functionObjects/forces" strathForces_OWN_INC)
hystrath_of_include_dirs(strathForces_OF_INC
  fileFormats transportModels transportModels/compressible
  TurbulenceModels/turbulenceModels TurbulenceModels/incompressible TurbulenceModels/compressible
  finiteVolume meshTools
)

add_library(strathForces SHARED
  ${HY}/src/functionObjects/forces/forces/forces.C
  ${HY}/src/functionObjects/forces/forceCoeffs/forceCoeffs.C
)
target_include_directories(strathForces PUBLIC
  ${strathForces_OWN_INC} ${strathForces_OF_INC}
  ${HY}/src/thermophysicalModels/strath/strathReactionThermo
)
target_link_libraries(strathForces PUBLIC
  strathSpecie strathReactionThermophysicalModels
  compressibleTransportModels incompressibleTransportModels
  turbulenceModels incompressibleTurbulenceModels compressibleTurbulenceModels
  fileFormats finiteVolume meshTools
)

# ============================================================================
# 16. applications/solvers/compressible/hy2Foam/BCs  ->  libhyTwoFoam
# ============================================================================

hystrath_own_include_dirs("${HY}/applications/solvers/compressible/hy2Foam/BCs" hyTwoFoam_OWN_INC)
hystrath_of_include_dirs(hyTwoFoam_OF_INC finiteVolume thermophysicalModels/specie thermophysicalModels/basic)

add_library(hyTwoFoam SHARED
  ${HY}/applications/solvers/compressible/hy2Foam/BCs/mixedFixedValueSlip/mixedFixedValueSlipFvPatchFields.C
  ${HY}/applications/solvers/compressible/hy2Foam/BCs/U/nonEqMaxwellSlipUFvPatchVectorField.C
  ${HY}/applications/solvers/compressible/hy2Foam/BCs/T/nonEqSmoluchowskiJumpT/nonEqSmoluchowskiJumpTFvPatchScalarField.C
  ${HY}/applications/solvers/compressible/hy2Foam/BCs/T/nonEqSmoluchowskiJumpTv/nonEqSmoluchowskiJumpTvFvPatchScalarField.C
  ${HY}/applications/solvers/compressible/hy2Foam/BCs/T/nonEqSmoluchowskiJumpTvMix/nonEqSmoluchowskiJumpTvMixFvPatchScalarField.C
  ${HY}/applications/solvers/compressible/hy2Foam/BCs/T/nonEqPattersonJumpT/nonEqPattersonJumpTFvPatchScalarField.C
)
target_include_directories(hyTwoFoam PUBLIC ${hyTwoFoam_OWN_INC} ${hyTwoFoam_OF_INC})
target_link_libraries(hyTwoFoam PUBLIC
  strathSpecie strathReactionThermophysicalModels finiteVolume
)

# ============================================================================
# 17. applications/solvers/compressible/hy2Foam  ->  hy2Foam (executable)
# ============================================================================

hystrath_of_include_dirs(hy2Foam_OF_INC
  finiteVolume meshTools sampling ODE transportModels/compressible
  thermophysicalModels/specie TurbulenceModels/turbulenceModels TurbulenceModels/compressible
  fvOptions
)

add_executable(hy2Foam
  ${HY}/applications/solvers/compressible/hy2Foam/hy2Foam.C
)
target_include_directories(hy2Foam PRIVATE
  ${HY}/applications/solvers/compressible/hy2Foam/BCs
  ${hy2Foam_OF_INC}
  ${HY}/src/thermophysicalModels/strath/strathSpecie
  ${HY}/src/thermophysicalModels/strath/strathReactionThermo
  ${HY}/src/thermophysicalModels/strath/strathChemistryModel
  ${HY}/src/thermophysicalModels/strath/strathSpecie/transport/speciesDiffusion/diffusivityModels
  ${HY}/src/thermophysicalModels/strath/strathSpecie/transport/speciesDiffusion/multiSpeciesTransportModels
  ${HY}/src/thermophysicalModels/strath/strathSpecie/relaxationTimes/relaxationTimeModels
  ${HY}/src/thermophysicalModels/strath/strathSpecie/rarefied/rarefactionParameters
  ${HY}/src/thermophysicalModels/strath/strathSpecie/transport/mixingRules
  ${HY}/src/hTCModels
)
target_link_libraries(hy2Foam PRIVATE
  hyTwoFoam finiteVolume meshTools
  strathFvOptions fvOptions ODE
  compressibleTransportModels
  strathSpecie strathReactionThermophysicalModels strathChemistryModel
  diffusivityModels multiSpeciesTransportModels
  VTRelaxationModels VVRelaxationModels eVRelaxationModels relaxationTimeModels
  mfpModels rarefactionParameters mixingRules
  compressibleTurbulenceModels hTCModels
)

# ============================================================================
# 18. applications/solvers/compressible/hy2MhdFoam  ->  hy2MhdFoam (executable)
# ============================================================================

add_executable(hy2MhdFoam
  ${HY}/applications/solvers/compressible/hy2MhdFoam/hy2MhdFoam.C
)
target_include_directories(hy2MhdFoam PRIVATE
  ${HY}/applications/solvers/compressible/hy2Foam
  ${hy2Foam_OF_INC}
  ${HY}/src/thermophysicalModels/strath/strathSpecie
  ${HY}/src/thermophysicalModels/strath/strathReactionThermo
  ${HY}/src/thermophysicalModels/strath/strathChemistryModel
  ${HY}/src/thermophysicalModels/strath/strathSpecie/transport/speciesDiffusion/diffusivityModels
  ${HY}/src/thermophysicalModels/strath/strathSpecie/transport/speciesDiffusion/multiSpeciesTransportModels
  ${HY}/src/thermophysicalModels/strath/strathSpecie/relaxationTimes/relaxationTimeModels
  ${HY}/src/thermophysicalModels/strath/strathSpecie/rarefied/rarefactionParameters
  ${HY}/src/thermophysicalModels/strath/strathSpecie/transport/mixingRules
  ${HY}/src/hTCModels
  ${HY}/src/mhdModel
)
target_link_libraries(hy2MhdFoam PRIVATE
  hyTwoFoam finiteVolume meshTools
  strathFvOptions fvOptions ODE
  compressibleTransportModels
  strathSpecie strathReactionThermophysicalModels strathChemistryModel
  diffusivityModels multiSpeciesTransportModels
  VTRelaxationModels VVRelaxationModels eVRelaxationModels relaxationTimeModels
  mfpModels rarefactionParameters mixingRules
  compressibleTurbulenceModels hTCModels spbpuMhdModels
)

# ============================================================================
# 19. Mesh utilities (independent of the CFD physics above)
# ============================================================================

hystrath_of_include_dirs(makeAxialMesh_OF_INC dynamicMesh cfdTools meshTools)
add_executable(makeAxialMesh
  ${HY}/applications/utilities/mesh/generation/makeAxialMesh/makeAxialMesh.C
)
target_include_directories(makeAxialMesh PRIVATE ${makeAxialMesh_OF_INC})
target_link_libraries(makeAxialMesh PRIVATE dynamicMesh meshTools)

hystrath_own_include_dirs("${HY}/applications/utilities/mesh/generation/blockMeshDG/lib" blockMeshDG_OWN_INC)
hystrath_of_include_dirs(blockMeshDG_OF_INC meshTools dynamicMesh)

add_library(blockMeshDG SHARED
  ${HY}/applications/utilities/mesh/generation/blockMeshDG/lib/curvedEdges/BSpline.C
  ${HY}/applications/utilities/mesh/generation/blockMeshDG/lib/curvedEdges/CatmullRomSpline.C
  ${HY}/applications/utilities/mesh/generation/blockMeshDG/lib/curvedEdges/polyLine.C
  ${HY}/applications/utilities/mesh/generation/blockMeshDG/lib/curvedEdges/arcEdge.C
  ${HY}/applications/utilities/mesh/generation/blockMeshDG/lib/curvedEdges/curvedEdge.C
  ${HY}/applications/utilities/mesh/generation/blockMeshDG/lib/curvedEdges/lineEdge.C
  ${HY}/applications/utilities/mesh/generation/blockMeshDG/lib/curvedEdges/polyLineEdge.C
  ${HY}/applications/utilities/mesh/generation/blockMeshDG/lib/curvedEdges/lineDivide.C
  ${HY}/applications/utilities/mesh/generation/blockMeshDG/lib/curvedEdges/splineEdge.C
  ${HY}/applications/utilities/mesh/generation/blockMeshDG/lib/blockDescriptor/blockDescriptor.C
  ${HY}/applications/utilities/mesh/generation/blockMeshDG/lib/blockDescriptor/blockDescriptorEdges.C
  ${HY}/applications/utilities/mesh/generation/blockMeshDG/lib/block/block.C
  ${HY}/applications/utilities/mesh/generation/blockMeshDG/lib/block/blockCreate.C
  ${HY}/applications/utilities/mesh/generation/blockMeshDG/lib/blockMesh/blockMesh.C
  ${HY}/applications/utilities/mesh/generation/blockMeshDG/lib/blockMesh/blockMeshCreate.C
  ${HY}/applications/utilities/mesh/generation/blockMeshDG/lib/blockMesh/blockMeshTopology.C
  ${HY}/applications/utilities/mesh/generation/blockMeshDG/lib/blockMesh/blockMeshCheck.C
  ${HY}/applications/utilities/mesh/generation/blockMeshDG/lib/blockMesh/blockMeshMerge.C
)
target_include_directories(blockMeshDG PUBLIC ${blockMeshDG_OWN_INC} ${blockMeshDG_OF_INC})
target_link_libraries(blockMeshDG PUBLIC meshTools dynamicMesh)

add_executable(blockMeshDGApp
  ${HY}/applications/utilities/mesh/generation/blockMeshDG/blockMeshApp.C
)
set_target_properties(blockMeshDGApp PROPERTIES OUTPUT_NAME blockMeshDG)
target_include_directories(blockMeshDGApp PRIVATE ${blockMeshDG_OF_INC})
target_link_libraries(blockMeshDGApp PRIVATE blockMeshDG meshTools dynamicMesh)

# ============================================================================
# 20. Summary
# ============================================================================

message(STATUS "Configured hyStrath targets:")
message(STATUS "  Libraries: strathSpecie strathReactionThermophysicalModels strathChemistryModel")
message(STATUS "             VTRelaxationModels VVRelaxationModels eVRelaxationModels relaxationTimeModels")
message(STATUS "             mfpModels rarefactionParameters mixingRules diffusivityModels multiSpeciesTransportModels")
message(STATUS "             strathFiniteVolume strathTurbulenceModels strathFvOptions hTCModels")
message(STATUS "             conductivityModels spbpuMhdModels strathFieldFunctionObjects strathForces")
message(STATUS "             hyTwoFoam blockMeshDG")
message(STATUS "  Solvers  : hy2Foam hy2MhdFoam")
message(STATUS "  Utilities: makeAxialMesh blockMeshDG (app)")
