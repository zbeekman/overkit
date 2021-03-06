# Copyright (c) 2018 Matthew J. Smith and Overkit contributors
# License: MIT (http://opensource.org/licenses/MIT)

#================
# Initialization
#================

set(GLOBAL_WARNING_FLAG FALSE CACHE INTERNAL "")

#====================
# CMake requirements
#====================

cmake_minimum_required(VERSION 2.8.12)

if(${CMAKE_VERSION} VERSION_GREATER 3.4.1)
  include(CheckFortranCompilerFlag)
  set(CMAKE_Fortran_CHECK_FLAG TRUE)
else()
  set(CMAKE_Fortran_CHECK_FLAG FALSE)
endif()

#===============
# Project setup
#===============

project(Overkit Fortran C)
set(Overkit_VERSION_MAJOR 1)
set(Overkit_VERSION_MINOR 0)
set(Overkit_VERSION_PATCH 4)
set(Overkit_VERSION ${Overkit_VERSION_MAJOR}.${Overkit_VERSION_MINOR}.${Overkit_VERSION_PATCH})

# Module stuff
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${CMAKE_SOURCE_DIR}/config/modules")
set(Fortran_ENABLED TRUE)
set(C_ENABLED TRUE)
set(CXX_ENABLED FALSE)

# OS X RPATH for dynamic libraries
set(CMAKE_MACOSX_RPATH TRUE)

# Add install lib dir to any executables' RPATH, as well as any paths to external libraries
set(CMAKE_INSTALL_RPATH "${CMAKE_INSTALL_PREFIX}/lib")
set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)

# Forbid in-source builds
if("${CMAKE_SOURCE_DIR}" STREQUAL "${CMAKE_BINARY_DIR}")
  message(FATAL_ERROR "In-source builds are not allowed. Please run CMake from a separate directory (you may need to remove CMakeFiles and CMakeCache.txt).")
endif()

# Adds "make distclean" command to delete contents of build directory (safer than rm -rf *)
add_custom_target(distclean COMMAND "${CMAKE_COMMAND}" -DBINARY_DIR="${CMAKE_BINARY_DIR}"
  -P "${CMAKE_SOURCE_DIR}/config/scripts/distclean.cmake")

#=========================
# Build types and options
#=========================

# Build types
include(DebugBuildType)
include(ReleaseBuildType)
include(CoverageBuildType)
if(NOT CMAKE_CONFIGURATION_TYPES)
  # Single-configuration generator
  if(CMAKE_BUILD_TYPE)
    set(BUILD_TYPE_VALUE ${CMAKE_BUILD_TYPE})
  else()
    message(STATUS "No build type specified; setting to 'Release'.")
    set(BUILD_TYPE_VALUE Release)
  endif()
  set(CMAKE_BUILD_TYPE ${BUILD_TYPE_VALUE} CACHE STRING
    "Choose the type of build, options are: None Debug Release Coverage." FORCE)
else()
  # Multi-configuration generator
  # (Not sure what to do here)
endif()

# Build options
option(TESTS "Enable tests" OFF)
option(EXAMPLES "Build examples" OFF)
option(BUILD_SHARED_LIBS "" OFF)
option(OPENMP "Enable thread parallelism via OpenMP" OFF)

if(NOT DEFINED BUILD_SHARED_LIBS)
  set(BUILD_SHARED_LIBS FALSE)
endif()

# OpenMP
if(OPENMP)
  find_package(OpenMP REQUIRED)
  if(NOT DEFINED OpenMP_Fortran_FLAGS)
    message(STATUS "OpenMP Fortran flags not defined; using C flags")
    set(OpenMP_Fortran_FLAGS "${OpenMP_C_FLAGS}")
  endif()
  # Need to specify -fopenmp/-qopenmp/etc. at link time too, but FindOpenMP doesn't
  # create a flag for it for some reason
  if(NOT OpenMP_Fortran_LIB_NAMES AND NOT OpenMP_C_LIB_NAMES)
    set(OpenMP_EXE_LINKER_FLAGS "${OpenMP_C_FLAGS}")
  endif()
endif()

#======================
# Extra compiler flags
#======================

set(EXTRA_Fortran_FLAGS "")
set(EXTRA_C_FLAGS "")

# GNU defaults to heap-allocating local arrays that have a size given by a function parameter.
# These are used all over the place, but only for small arrays, so they should be
# stack-allocated instead to avoid massive performance hit
# Note: this is only supported in GCC 4.7 or newer
if(${CMAKE_Fortran_COMPILER_ID} MATCHES "GNU")
  if(CMAKE_Fortran_CHECK_FLAG)
    unset(SUPPORTS_FSTACKARRAYS CACHE)
    CHECK_Fortran_COMPILER_FLAG(-fstack-arrays SUPPORTS_FSTACKARRAYS)
  else()
    if(CMAKE_C_COMPILER_VERSION VERSION_GREATER 4.7)
      set(SUPPORTS_FSTACKARRAYS TRUE CACHE INTERNAL "")
    else()
      set(SUPPORTS_FSTACKARRAYS FALSE CACHE INTERNAL "")
    endif()
  endif()
  if(SUPPORTS_FSTACKARRAYS)
    set(EXTRA_Fortran_FLAGS "${EXTRA_Fortran_FLAGS} -fstack-arrays")
  else()
    message(WARNING "GNU Fortran compiler version doesn't support -fstack-arrays flag. Performance may be degraded.")
    set(GLOBAL_WARNING_FLAG TRUE CACHE INTERNAL "")
  endif()
endif()

# Intel sometimes allocates large arrays on the stack; this flag prevents
# that for arrays larger than 100KB
if(${CMAKE_Fortran_COMPILER_ID} MATCHES "Intel")
  set(EXTRA_Fortran_FLAGS "${EXTRA_Fortran_FLAGS} -heap-arrays 100")
endif()

set(CMAKE_Fortran_FLAGS_DEBUG "${CMAKE_Fortran_FLAGS_DEBUG} ${EXTRA_Fortran_FLAGS}" CACHE STRING
  "Flags used by the Fortran compiler during debug builds." FORCE)
set(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} ${EXTRA_C_FLAGS}" CACHE STRING
  "Flags used by the C compiler during debug builds." FORCE)
set(CMAKE_Fortran_FLAGS_RELEASE "${CMAKE_Fortran_FLAGS_RELEASE} ${EXTRA_Fortran_FLAGS}" CACHE STRING
  "Flags used by the Fortran compiler during release builds." FORCE)
set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} ${EXTRA_C_FLAGS}" CACHE STRING
  "Flags used by the C compiler during release builds." FORCE)
set(CMAKE_Fortran_FLAGS_COVERAGE "${CMAKE_Fortran_FLAGS_COVERAGE} ${EXTRA_Fortran_FLAGS}" CACHE STRING
  "Flags used by the Fortran compiler during coverage builds." FORCE)
set(CMAKE_C_FLAGS_COVERAGE "${CMAKE_C_FLAGS_COVERAGE} ${EXTRA_C_FLAGS}" CACHE STRING
  "Flags used by the C compiler during coverage builds." FORCE)

#======================
# Testing and coverage
#======================

# Enable testing
if(TESTS)
  enable_testing()
  find_package(pFUnit)
  if(NOT PFUNIT_FOUND)
    message(WARNING "Could not find pFUnit. Unit tests will not be built.")
    set(GLOBAL_WARNING_FLAG TRUE CACHE INTERNAL "")
  endif()
endif()

# Verbose version of 'make test'
add_custom_target(test-v COMMAND ${CMAKE_CTEST_COMMAND} --verbose)

# Coverage report
add_custom_target(coverage
  COMMAND "${CMAKE_COMMAND}" -DSOURCE_DIR="${CMAKE_SOURCE_DIR}" -DBINARY_DIR="${CMAKE_BINARY_DIR}"
    -DCOVERAGE_DIR="src/core" -P "${CMAKE_SOURCE_DIR}/config/scripts/gen-coverage-data.cmake"
  COMMAND "${CMAKE_COMMAND}" -DSOURCE_DIR="${CMAKE_SOURCE_DIR}" -DBINARY_DIR="${CMAKE_BINARY_DIR}"
    -DCOVERAGE_DIR="src/support" -P "${CMAKE_SOURCE_DIR}/config/scripts/gen-coverage-data.cmake"
  COMMAND "${CMAKE_COMMAND}" -DSOURCE_DIR="${CMAKE_SOURCE_DIR}" -DBINARY_DIR="${CMAKE_BINARY_DIR}"
    -DCOVERAGE_DIR="src/utils" -P "${CMAKE_SOURCE_DIR}/config/scripts/gen-coverage-data.cmake"
  COMMAND "${CMAKE_COMMAND}" -DSOURCE_DIR="${CMAKE_SOURCE_DIR}" -DBINARY_DIR="${CMAKE_BINARY_DIR}"
    -P "${CMAKE_SOURCE_DIR}/config/scripts/gen-coverage-report.cmake"
)

# Reset coverage data
add_custom_target(coverageclean
  COMMAND "${CMAKE_COMMAND}" -DBINARY_DIR="${CMAKE_BINARY_DIR}"
    -P "${CMAKE_SOURCE_DIR}/config/scripts/coverageclean.cmake"
)

#=====================
# Other configuration
#=====================

# Shut ranlib up about no symbols
if(APPLE)
  set(CMAKE_C_ARCHIVE_CREATE "<CMAKE_AR> Scr <TARGET> <LINK_FLAGS> <OBJECTS>")
  set(CMAKE_CXX_ARCHIVE_CREATE "<CMAKE_AR> Scr <TARGET> <LINK_FLAGS> <OBJECTS>")
  set(CMAKE_Fortran_ARCHIVE_CREATE "<CMAKE_AR> Scr <TARGET> <LINK_FLAGS> <OBJECTS>")
  set(CMAKE_C_ARCHIVE_FINISH "<CMAKE_RANLIB> -no_warning_for_no_symbols -c <TARGET>")
  set(CMAKE_CXX_ARCHIVE_FINISH "<CMAKE_RANLIB> -no_warning_for_no_symbols -c <TARGET>")
  set(CMAKE_Fortran_ARCHIVE_FINISH "<CMAKE_RANLIB> -no_warning_for_no_symbols -c <TARGET>")
endif()

#================
# Subdirectories
#================

add_subdirectory(examples)
add_subdirectory(src)
add_subdirectory(tests)

#==============
# Installation
#==============

include(CMakePackageConfigHelpers)

write_basic_package_version_file("${CMAKE_BINARY_DIR}/OverkitConfigVersion.cmake"
  VERSION ${Overkit_VERSION}
  COMPATIBILITY SameMajorVersion
)
if(CMAKE_MAJOR_VERSION GREATER 3)
  export(EXPORT OverkitTargets FILE "${CMAKE_BINARY_DIR}/OverkitTargets.cmake")
else()
  export(TARGETS overkit FILE "${CMAKE_BINARY_DIR}/OverkitTargets.cmake")
endif()

configure_file(config/OverkitConfig.cmake
  "${CMAKE_BINARY_DIR}/OverkitConfig.cmake"
  COPYONLY
)

install(
  FILES
    "${CMAKE_BINARY_DIR}/OverkitConfig.cmake"
    "${CMAKE_BINARY_DIR}/OverkitConfigVersion.cmake"
  DESTINATION lib/Overkit/CMake
)

install(EXPORT OverkitTargets
  DESTINATION lib/Overkit/CMake
)

#=============================
# Write configuration summary
#=============================

message(STATUS "")
message(STATUS "=====================")
message(STATUS "Configuration Summary")
message(STATUS "=====================")
message(STATUS "")
message(STATUS "Build type:         ${CMAKE_BUILD_TYPE}")
message(STATUS "Fortran compiler:   ${CMAKE_Fortran_COMPILER}")
message(STATUS "C compiler:         ${CMAKE_C_COMPILER}")
message(STATUS "Shared libraries:   ${BUILD_SHARED_LIBS}")
message(STATUS "Install prefix:     ${CMAKE_INSTALL_PREFIX}")
message(STATUS "Examples:           ${EXAMPLES}")
message(STATUS "Tests:              ${TESTS}")
message(STATUS "OpenMP support:     ${OPENMP}")
message(STATUS "")
if(GLOBAL_WARNING_FLAG)
  message(STATUS "**** Warnings were produced. See details above. **** ")
  message(STATUS "")
endif()
