# reflect-sm/CMakeLists.txt

cmake_minimum_required(VERSION 3.10)

project(reflect-sm VERSION 0.1)
enable_language(CXX)

#include(cmake/cxx.cmake)
#include(cmake/code-coverage.cmake)

# ----------------------------------------------------------------
# unit test setup

enable_testing()

###  copy from proj/reflect/CMakeLists.txt

# ----------------------------------------------------------------
# c++ settings

set(XO_PROJECT_NAME reflect-sm)
set(PROJECT_CXX_FLAGS "")

add_definitions(${PROJECT_CXX_FLAGS})

if(NOT CMAKE_CXX_STANDARD)
    set(CMAKE_CXX_STANDARD 20)
endif()

set(CMAKE_CXX_STANDARD_REQUIRED True)

# always write compile_commands.json
# (this does not seem to apply to external projects)
#
set(CMAKE_EXPORT_COMPILE_COMMANDS ON CACHE INTERNAL "")

if(NOT CMAKE_INSTALL_RPATH)
    set(CMAKE_INSTALL_RPATH ${CMAKE_INSTALL_PREFIX}/lib CACHE STRING "runpath in installed libraries/executables")
endif()

message("CMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}")
message("CMAKE_INSTALL_RPATH=${CMAKE_INSTALL_RPATH}")

# ----------------------------------------------------------------
# external projects (need these to exist before add_subdirectory() below)
#
# we are expecting these projects to coexist peacefully when installed to
# ${PROJECT_BINARY_DIR}/local (e.g. ./build/local).
# in other words: ther `make install` steps can be run:
# - all with the same CMAKE_INSTALL_PREFIX
# - without causing filesystem collisions / other failures
# - provided in bottom-up dependency order,
#   i.e. depend-ees installed before depend-ers
#
# NOTES:
#
# Monorepo vs Manyrepo
# 1. we want to use git submodules for these dependencies,  so we can
#    recruit git to (perhaps modestly) help streamline coordinated commits.
# 2. monorepo problematic because we have some very loosely coupled libraries
#    (e.g. reflect + process + websocket);  don't want to permanently
#    weld the codebases together
# 3. separate + independent repos,  all installing to common location (say /usr/local)
#    is simple + efficient.   Requires always building in bottom-up dependency order,
#    in which case everything is built just once.
#    However it's fragile and error-prone,  since there's nothing
#    to ensure that libraries + their dependencies are consistent.
#    Easy to inadvertently fail to build an upstream dependency,  which is a train to
#    undefined behavior.
# 4. git submodules intend to help with this,  but don't solve for diamond dependencies.
#    if we have repo deps
#      A->{B,C}->D
#    and {A,B,C} use submodules,  then {B,C} will host their own clones of D.
#    Rejecting this as not scaling well.
# 5. instead provide codebases {A,B,C,D} without submodules,
#    and scaffold submodules into separate repos {A',B',C'};
#    where:
#    - C' has submodules {C,D}
#    - B' has submodules {B,D}
#    - A' has submodules {A,B,C,D}
#    This is such a scaffolding repo.
#    The cmake files in a scaffolding repo guarantee a consistent build.
#    Cost: build in A' can't share any build artifacts with build in
#    another repo B'.
# 6. for building/installing, we do provide nix integration,  see for example the xo-nix repo
#    (https://github.com/Rconybea/xo-nix.git).
#    Selling point here is that nix gives us accurate dependency tracking,
#    optimal build time (each codebase built once),  and easy to maintain.
#    Main liability is it doesn't directly help with coordinated commits.
#    (though want to reasses w/ nix flakes)
#
# Runpath:
# 1. We build dependencies with INSTALL_DIR -> ${PROJECT_BINARY_DIR}/local.
# 2. For install to ${CMAKE_INSTALL_PREFIX} we generally just need to overlay ${PROJECT_BINARY_DIR}/local
#    onto ${CMAKE_INSTALL_PREFIX};  e.g. everything in ${PROJECT_BINARY_DIR}/local/include should get
#    copied to corresponding location under ${CMAKE_INSTALL_PREFIX}/include
# 3. To use this strategy for {shared libraries + executables},  we need their DT_RUNPATH entries
#    to refer to ${CMAKE_INSTALL_PREFIX}/lib,  not ${PROJECT_BINARY_DIR}/lib.
# 4. This is accomplished by setting ${CMAKE_INSTALL_RPATH} above,  and forwarding it to
#    each external-project build below
# 5. It follows that to use libraries + executables in ${PROJECT_BINARY_DIR}/local (before final install),
#    we'll need an assist from LD_LIBRARY_PATH:
#      LD_LIBRARY_PATH=${PROJECT_BINARY_DIR}/lib
# 6. Note that this ducking/weaving is unique to submodule repos like this one:
#    nix takes responsibility for rewriting RUNPATH entries in repos like xo-nix.

include(ExternalProject)

# NOTE:  we could have cmake handle git interaction,
#        but we want source for certain dependencies to live in a location
#        that's suitable for accepting changes + coordinated commits.
#        In particular,  not in the build directory!
#

## ----- xo-cmake -----

externalproject_add(
    project_xo_cmake
    SOURCE_DIR ${PROJECT_SOURCE_DIR}/repo/xo-cmake
    BINARY_DIR ${PROJECT_BINARY_DIR}/ext/xo-cmake
    INSTALL_DIR ${PROJECT_BINARY_DIR}/local
    CMAKE_ARGS -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE} -DCODE_COVERAGE=${CODE_COVERAGE} -DCMAKE_PREFIX_PATH=<INSTALL_DIR> -DCMAKE_INSTALL_RPATH=${CMAKE_INSTALL_RPATH} -DCMAKE_INSTALL_PREFIX=<INSTALL_DIR>
    BUILD_COMMAND make
    INSTALL_COMMAND make install
    TEST_BEFORE_INSTALL False
)
set_target_properties(
    project_xo_cmake
    PROPERTIES ADDITIONAL_CLEAN_FILES "${PROJECT_BINARY_DIR}/ext/xo-cmake;${PROJECT_BINARY_DIR}/local/share/cmake/xo_macros"
)

# projects that use xo-cmake macros will need arg like -DCMAKE_MODULE_PATH=${PROJECT_BINARY_DIR}/local/share/cmake to cmake configure step

## ----- indentlog ------

externalproject_add(
    project_indentlog
    SOURCE_DIR ${PROJECT_SOURCE_DIR}/repo/indentlog
    BINARY_DIR ${PROJECT_BINARY_DIR}/ext/indentlog
    INSTALL_DIR ${PROJECT_BINARY_DIR}/local
    CMAKE_ARGS -DCMAKE_MODULE_PATH=<INSTALL_DIR>/share/cmake -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE} -DCODE_COVERAGE=${CODE_COVERAGE} -DCMAKE_PREFIX_PATH=<INSTALL_DIR> -DCMAKE_INSTALL_RPATH=${CMAKE_INSTALL_RPATH} -DCMAKE_INSTALL_PREFIX=<INSTALL_DIR>
    BUILD_COMMAND make
    INSTALL_COMMAND make install
    TEST_BEFORE_INSTALL True
)
set_target_properties(
    project_indentlog
    PROPERTIES ADDITIONAL_CLEAN_FILES "${PROJECT_BINARY_DIR}/ext/indentlog;${PROJECT_BINARY_DIR}/local/include/xo/indentlog;${PROJECT_BINARY_DIR}/local/bin/indentlog;${PROJECT_BINARY_DIR}/local/lib/cmake/indentlog"
)

add_library(indentlog INTERFACE IMPORTED)
#set_property(TARGET indentlog PROPERTY IMPORTED_LOCATION ${PROJECT_BINARY_DIR}/local/lib/libindentlog.so)
add_dependencies(project_indentlog project_xo_cmake)
add_dependencies(indentlog project_indentlog)

# runs ctest in indentlog build dir
add_test(NAME indentlog COMMAND ${PROJECT_SOURCE_DIR}/cmake/run-external-ctest ${PROJECT_BINARY_DIR}/ext/indentlog)
#target_code_coverage(indentlog EXTERNAL AUTO ALL)

# ----- subsys -----

externalproject_add(
    project_subsys
    SOURCE_DIR ${PROJECT_SOURCE_DIR}/repo/subsys
    BINARY_DIR ${PROJECT_BINARY_DIR}/ext/subsys
    INSTALL_DIR ${PROJECT_BINARY_DIR}/local
    CMAKE_ARGS -DCMAKE_MODULE_PATH=<INSTALL_DIR>/share/cmake -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE} -DCODE_COVERAGE=${CODE_COVERAGE} -DCMAKE_PREFIX_PATH=<INSTALL_DIR> -DCMAKE_INSTALL_RPATH=${CMAKE_INSTALL_RPATH} -DCMAKE_INSTALL_PREFIX=<INSTALL_DIR>
    BUILD_COMMAND make
    INSTALL_COMMAND make install
    TEST_BEFORE_INSTALL True
)
set_target_properties(
    project_subsys
    PROPERTIES ADDITIONAL_CLEAN_FILES "${PROJECT_BINARY_DIR}/ext/subsys;${PROJECT_BINARY_DIR}/local/include/xo/subsys;${PROJECT_BINARY_DIR}/local/lib/cmake/subsys"
)

add_library(subsys INTERFACE IMPORTED)
add_dependencies(project_subsys project_xo_cmake)
add_dependencies(subsys project_subsys)

# runs ctest in subsys build dir
add_test(NAME subsys COMMAND ${PROJECT_SOURCE_DIR}/cmake/run-external-ctest ${PROJECT_BINARY_DIR}/ext/subsys)

# ----- refcnt -----

# CMAKE_ARGS
#   CMAKE_BUILD_TYPE propagate Debug/Release build type
#   CODE_COVERAGE propagate code coverage setting
#   CMAKE_PREFIX_PATH path for support cmake files of dependencies (needed for find_package() to work)
#   CMAKE_INSTALL_RPATH include in DT_RUNPATH entries for libraries+executables.   Not INSTALL_DIR
#   CMAKE_INSTALL_PREFIX install subproject here
# SOURCE_DIR  -- where to find already established source code
# BINARY_DIR  -- run build for external project here
# INSTALL_DIR -- (temporarily) install external project here
#
externalproject_add(
    project_refcnt
    SOURCE_DIR ${PROJECT_SOURCE_DIR}/repo/refcnt
    BINARY_DIR ${PROJECT_BINARY_DIR}/ext/refcnt
    INSTALL_DIR ${PROJECT_BINARY_DIR}/local
    CMAKE_ARGS -DCMAKE_MODULE_PATH=<INSTALL_DIR>/share/cmake -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE} -DCODE_COVERAGE=${CODE_COVERAGE} -DCMAKE_PREFIX_PATH=<INSTALL_DIR> -DCMAKE_INSTALL_RPATH=${CMAKE_INSTALL_RPATH} -DCMAKE_INSTALL_PREFIX=<INSTALL_DIR>
    BUILD_COMMAND make
    INSTALL_COMMAND make install
    TEST_BEFORE_INSTALL True
)
set_target_properties(
    project_refcnt
    PROPERTIES ADDITIONAL_CLEAN_FILES "${PROJECT_BINARY_DIR}/ext/refcnt;${PROJECT_BINARY_DIR}/local/include/xo/refcnt;${PROJECT_BINARY_DIR}/local/include/xo/cxxutil;${PROJECT_BINARY_DIR}/local/lib/cmake/refcnt"
)

add_library(refcnt SHARED IMPORTED)
set_property(TARGET refcnt PROPERTY IMPORTED_LOCATION ${PROJECT_BINARY_DIR}/local/lib/librefcnt.so)
add_dependencies(refcnt project_refcnt)
add_dependencies(project_refcnt project_indentlog)

# runs ctest in refcnt build dir
add_test(NAME refcnt COMMAND ${PROJECT_SOURCE_DIR}/cmake/run-external-ctest ${PROJECT_BINARY_DIR}/ext/refcnt)

# ----- reflect -----

# CMAKE_ARGS
#   CMAKE_BUILD_TYPE propagate Debug/Release build type
#   CODE_COVERAGE propagate code coverage setting
#   CMAKE_PREFIX_PATH path for support cmake files of dependencies (needed for find_package() to work)
#   CMAKE_INSTALL_PREFIX install subproject here
# SOURCE_DIR  -- where to find already established source code
# BINARY_DIR  -- run build for external project here
# INSTALL_DIR -- (temporarily) install external project here
#
externalproject_add(
    project_reflect
    SOURCE_DIR ${PROJECT_SOURCE_DIR}/repo/reflect
    BINARY_DIR ${PROJECT_BINARY_DIR}/ext/reflect
    INSTALL_DIR ${PROJECT_BINARY_DIR}/local
    CMAKE_ARGS -DCMAKE_MODULE_PATH=<INSTALL_DIR>/share/cmake -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE} -DCODE_COVERAGE=${CODE_COVERAGE} -DCMAKE_PREFIX_PATH=<INSTALL_DIR> -DCMAKE_INSTALL_RPATH=${CMAKE_INSTALL_RPATH} -DCMAKE_INSTALL_PREFIX=<INSTALL_DIR>
    BUILD_COMMAND make
    INSTALL_COMMAND make install
    TEST_BEFORE_INSTALL True
)
set_target_properties(
    project_reflect
    PROPERTIES ADDITIONAL_CLEAN_FILES "${PROJECT_BINARY_DIR}/ext/reflect;${PROJECT_BINARY_DIR}/local/include/xo/reflect;${PROJECT_BINARY_DIR}/local/lib/cmake/reflect"
)

add_library(reflect SHARED IMPORTED)
set_property(TARGET reflect PROPERTY IMPORTED_LOCATION ${PROJECT_BINARY_DIR}/local/lib/libreflect.so)
add_dependencies(reflect project_reflect)
add_dependencies(project_reflect project_indentlog)
add_dependencies(project_reflect project_subsys)
add_dependencies(project_reflect project_refcnt)

# runs ctest in reflect build dir
add_test(NAME reflect COMMAND ${PROJECT_SOURCE_DIR}/cmake/run-external-ctest ${PROJECT_BINARY_DIR}/ext/reflect)

# ----- printjson -----

# CMAKE_ARGS
#   CMAKE_BUILD_TYPE propagate Debug/Release build type
#   CODE_COVERAGE propagate code coverage setting
#   CMAKE_PREFIX_PATH path for support cmake files of dependencies (needed for find_package() to work)
#   CMAKE_INSTALL_PREFIX install subproject here
# SOURCE_DIR  -- where to find already established source code
# BINARY_DIR  -- run build for external project here
# INSTALL_DIR -- (temporarily) install external project here
#
externalproject_add(
    project_printjson
    SOURCE_DIR ${PROJECT_SOURCE_DIR}/repo/xo-printjson
    BINARY_DIR ${PROJECT_BINARY_DIR}/ext/printjson
    INSTALL_DIR ${PROJECT_BINARY_DIR}/local
    CMAKE_ARGS -DCMAKE_MODULE_PATH=<INSTALL_DIR>/share/cmake -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE} -DCODE_COVERAGE=${CODE_COVERAGE} -DCMAKE_PREFIX_PATH=<INSTALL_DIR> -DCMAKE_INSTALL_RPATH=${CMAKE_INSTALL_RPATH} -DCMAKE_INSTALL_PREFIX=<INSTALL_DIR>
    BUILD_COMMAND make
    INSTALL_COMMAND make install
    TEST_BEFORE_INSTALL True
)
set_target_properties(
    project_printjson
    PROPERTIES ADDITIONAL_CLEAN_FILES "${PROJECT_BINARY_DIR}/ext/printjson;${PROJECT_BINARY_DIR}/local/include/xo/printjson;${PROJECT_BINARY_DIR}/local/lib/cmake/printjson"
)

add_library(printjson SHARED IMPORTED)
set_property(TARGET printjson PROPERTY IMPORTED_LOCATION ${PROJECT_BINARY_DIR}/local/lib/libprintjson.so)
add_dependencies(printjson project_printjson)
add_dependencies(project_printjson project_reflect)

# runs ctest in printjson build dir
add_test(NAME printjson COMMAND ${PROJECT_SOURCE_DIR}/cmake/run-external-ctest ${PROJECT_BINARY_DIR}/ext/printjson)

# ----- randomgen -----

# CMAKE_ARGS
#   CMAKE_BUILD_TYPE propagate Debug/Release build type
#   CODE_COVERAGE propagate code coverage setting
#   CMAKE_PREFIX_PATH path for support cmake files of dependencies (needed for find_package() to work)
#   CMAKE_INSTALL_PREFIX install subproject here
# SOURCE_DIR  -- where to find already established source code
# BINARY_DIR  -- run build for external project here
# INSTALL_DIR -- (temporarily) install external project here
#
externalproject_add(
    project_randomgen
    SOURCE_DIR ${PROJECT_SOURCE_DIR}/repo/randomgen
    BINARY_DIR ${PROJECT_BINARY_DIR}/ext/randomgen
    INSTALL_DIR ${PROJECT_BINARY_DIR}/local
    CMAKE_ARGS -DCMAKE_MODULE_PATH=<INSTALL_DIR>/share/cmake -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE} -DCODE_COVERAGE=${CODE_COVERAGE} -DCMAKE_PREFIX_PATH=<INSTALL_DIR> -DCMAKE_INSTALL_RPATH=${CMAKE_INSTALL_RPATH} -DCMAKE_INSTALL_PREFIX=<INSTALL_DIR>
    BUILD_COMMAND make
    INSTALL_COMMAND make install
    TEST_BEFORE_INSTALL True
)
set_target_properties(
    project_randomgen
    PROPERTIES ADDITIONAL_CLEAN_FILES "${PROJECT_BINARY_DIR}/bin/randomgen;${PROJECT_BINARY_DIR}/ext/randomgen;${PROJECT_BINARY_DIR}/local/include/xo/randomgen;${PROJECT_BINARY_DIR}/local/lib/cmake/randomgen"
)

add_library(randomgen SHARED IMPORTED)
#set_property(TARGET randomgen PROPERTY IMPORTED_LOCATION ${PROJECT_BINARY_DIR}/local/lib/librandomgen.so)
add_dependencies(randomgen project_randomgen)
add_dependencies(project_randomgen project_indentlog)
add_dependencies(project_randomgen project_xo_cmake)
#add_dependencies(project_reflect project_subsys)

# runs ctest in randomgen build dir
#add_test(NAME randomgen COMMAND ${PROJECT_SOURCE_DIR}/cmake/run-external-ctest ${PROJECT_BINARY_DIR}/ext/randomgen)

# ----- ordinaltree -----

# CMAKE_ARGS
#   CMAKE_BUILD_TYPE propagate Debug/Release build type
#   CODE_COVERAGE propagate code coverage setting
#   CMAKE_PREFIX_PATH path for support cmake files of dependencies (needed for find_package() to work)
#   CMAKE_INSTALL_PREFIX install subproject here
# SOURCE_DIR  -- where to find already established source code
# BINARY_DIR  -- run build for external project here
# INSTALL_DIR -- (temporarily) install external project here
#
externalproject_add(
    project_ordinaltree
    SOURCE_DIR ${PROJECT_SOURCE_DIR}/repo/xo-ordinaltree
    BINARY_DIR ${PROJECT_BINARY_DIR}/ext/ordinaltree
    INSTALL_DIR ${PROJECT_BINARY_DIR}/local
    CMAKE_ARGS -DCMAKE_MODULE_PATH=<INSTALL_DIR>/share/cmake -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE} -DCODE_COVERAGE=${CODE_COVERAGE} -DCMAKE_PREFIX_PATH=<INSTALL_DIR> -DCMAKE_INSTALL_RPATH=${CMAKE_INSTALL_RPATH} -DCMAKE_INSTALL_PREFIX=<INSTALL_DIR>
    BUILD_COMMAND make
    INSTALL_COMMAND make install
    TEST_BEFORE_INSTALL True
)
set_target_properties(
    project_ordinaltree
    PROPERTIES ADDITIONAL_CLEAN_FILES "${PROJECT_BINARY_DIR}/ext/ordinaltree;${PROJECT_BINARY_DIR}/local/include/xo/ordinaltree;${PROJECT_BINARY_DIR}/local/lib/cmake/xo_ordinaltree"
)

add_library(ordinaltree SHARED IMPORTED)
set_property(TARGET ordinaltree PROPERTY IMPORTED_LOCATION ${PROJECT_BINARY_DIR}/local/lib/libordinaltree.so)
add_dependencies(ordinaltree project_ordinaltree)
add_dependencies(project_ordinaltree project_randomgen)
add_dependencies(project_ordinaltree project_refcnt)

# runs ctest in ordinaltree build dir
add_test(NAME ordinaltree COMMAND ${PROJECT_SOURCE_DIR}/cmake/run-external-ctest ${PROJECT_BINARY_DIR}/ext/ordinaltree)

# ----- xo-pyutil -----

# CMAKE_ARGS
#   CMAKE_BUILD_TYPE propagate Debug/Release build type
#   CODE_COVERAGE propagate code coverage setting
#   CMAKE_PREFIX_PATH path for support cmake files of dependencies (needed for find_package() to work)
#   CMAKE_INSTALL_PREFIX install subproject here
# SOURCE_DIR  -- where to find already established source code
# BINARY_DIR  -- run build for external project here
# INSTALL_DIR -- (temporarily) install external project here
#
externalproject_add(
    project_pyutil
    SOURCE_DIR ${PROJECT_SOURCE_DIR}/repo/xo-pyutil
    BINARY_DIR ${PROJECT_BINARY_DIR}/ext/pyutil
    INSTALL_DIR ${PROJECT_BINARY_DIR}/local
    CMAKE_ARGS -DCMAKE_MODULE_PATH=<INSTALL_DIR>/share/cmake -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE} -DCODE_COVERAGE=${CODE_COVERAGE} -DCMAKE_PREFIX_PATH=<INSTALL_DIR> -DCMAKE_INSTALL_RPATH=${CMAKE_INSTALL_RPATH} -DCMAKE_INSTALL_PREFIX=<INSTALL_DIR>
    BUILD_COMMAND make
    INSTALL_COMMAND make install
    TEST_BEFORE_INSTALL True
)
set_target_properties(
    project_pyutil
    PROPERTIES ADDITIONAL_CLEAN_FILES "${PROJECT_BINARY_DIR}/ext/pyutil;${PROJECT_BINARY_DIR}/local/include/xo/pyutil;${PROJECT_BINARY_DIR}/local/lib/cmake/xo_pyutil"
)

add_library(pyutil SHARED IMPORTED)
#set_property(TARGET pyutil PROPERTY IMPORTED_LOCATION ${PROJECT_BINARY_DIR}/local/lib/libpyutil.so)
add_dependencies(pyutil project_pyutil)
add_dependencies(project_pyutil project_xo_cmake)
#add_dependencies(project_pyutil project_refcnt)

# runs ctest in pyutil build dir
#add_test(NAME pyutil COMMAND ${PROJECT_SOURCE_DIR}/cmake/run-external-ctest ${PROJECT_BINARY_DIR}/ext/pyutil)

# ----- xo-pyreflect -----

# CMAKE_ARGS
#   CMAKE_BUILD_TYPE propagate Debug/Release build type
#   CODE_COVERAGE propagate code coverage setting
#   CMAKE_PREFIX_PATH path for support cmake files of dependencies (needed for find_package() to work)
#   CMAKE_INSTALL_PREFIX install subproject here
# SOURCE_DIR  -- where to find already established source code
# BINARY_DIR  -- run build for external project here
# INSTALL_DIR -- (temporarily) install external project here
#
externalproject_add(
    project_pyreflect
    SOURCE_DIR ${PROJECT_SOURCE_DIR}/repo/xo-pyreflect
    BINARY_DIR ${PROJECT_BINARY_DIR}/ext/pyreflect
    INSTALL_DIR ${PROJECT_BINARY_DIR}/local
    CMAKE_ARGS -DCMAKE_MODULE_PATH=<INSTALL_DIR>/share/cmake -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE} -DCODE_COVERAGE=${CODE_COVERAGE} -DCMAKE_PREFIX_PATH=<INSTALL_DIR> -DCMAKE_INSTALL_RPATH=${CMAKE_INSTALL_RPATH} -DCMAKE_INSTALL_PREFIX=<INSTALL_DIR>
    BUILD_COMMAND make
    INSTALL_COMMAND make install
    TEST_BEFORE_INSTALL True
)
set_target_properties(
    project_pyreflect
    PROPERTIES ADDITIONAL_CLEAN_FILES "${PROJECT_BINARY_DIR}/ext/pyreflect;${PROJECT_BINARY_DIR}/local/include/xo/pyreflect;${PROJECT_BINARY_DIR}/local/lib/cmake/xo_pyreflect"
)

add_library(pyreflect SHARED IMPORTED)
#set_property(TARGET pyreflect PROPERTY IMPORTED_LOCATION ${PROJECT_BINARY_DIR}/local/lib/libpyreflect.so)
add_dependencies(pyreflect project_pyreflect)
add_dependencies(project_pyreflect project_xo_cmake)
add_dependencies(project_pyreflect project_reflect)

# runs ctest in pyreflect build dir
#add_test(NAME pyreflect COMMAND ${PROJECT_SOURCE_DIR}/cmake/run-external-ctest ${PROJECT_BINARY_DIR}/ext/pyreflect)
