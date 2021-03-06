cmake_minimum_required(VERSION 3.3)
project(SCALAPACK VERSION 2.8 LANGUAGES C Fortran)

message(STATUS "CMAKE_MODULE_PATH: ${CMAKE_MODULE_PATH}")
option(BUILD_TESTS "${PROJECT_NAME} - Build tests" ON)
set(PACKAGE_CONFIG_DIR "lib/cmake" CACHE STRING "Directory for package config files (relative to CMAKE_INSTALL_PREFIX)")
if (NOT DEFINED FORTRAN_MANGLING)
    message(STATUS "No fortran mangling known setting default to Add_.")
    set(FORTRAN_MANGLING Add_)
elseif (FORTRAN_MANGLING STREQUAL "NO")
    set(FORTRAN_MANGLING NoChange)
endif()
set(FORTRAN_MANGLING ${FORTRAN_MANGLING} CACHE STRING "${PROJECT_NAME} - Fortran mangling scheme")
set(BUILD_PRECISION sdcz CACHE STRING "${PROJECT_NAME} - Build precisions (sdcz)")

# Always build single precision (used by some tests [e.g. PETSc] and examples)
set(BUILD_PRECISION "s${BUILD_PRECISION}")

# Find BLAS separately so we can use 'blas' target.
find_package(BLAS ${BLAS_VERSION} REQUIRED)
find_package(LAPACK ${LAPACK_VERSION} REQUIRED)

#
#   MPI
#
find_package(MPI REQUIRED)

# Configure the warning and code coverage suppression file
configure_file( 
  "${SCALAPACK_SOURCE_DIR}/CMAKE/CTestCustom.cmake.in"
  "${SCALAPACK_BINARY_DIR}/CTestCustom.cmake"
  COPYONLY
)

# Add the CMake directory for custon CMake modules
set(CMAKE_MODULE_PATH "${SCALAPACK_SOURCE_DIR}/CMAKE" ${CMAKE_MODULE_PATH})

if (UNIX AND CMAKE_Fortran_COMPILER_ID MATCHES Intel)
  set(CMAKE_Fortran_FLAGS "${CMAKE_Fortran_FLAGS} -fltconsistency -fp_port" )
  # Strip the "warn all" flag on intel - it stops with an error :-)
  string(REPLACE "-warn all" " " CMAKE_Fortran_FLAGS_DEBUG "${CMAKE_Fortran_FLAGS_DEBUG}")
endif()

# --------------------------------------------------
# Testing
if (BUILD_TESTS)
    SET(DART_TESTING_TIMEOUT 600)
    enable_testing()
    include(CTest)
endif()

# --------------------------------------------------
# Organize output files.  On Windows this also keeps .dll files next
# to the .exe files that need them, making tests easy to run.
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${SCALAPACK_BINARY_DIR}/TESTING)

# --------------------------------------------------
# Check for any necessary platform specific compiler flags
include(CheckBLACSCompilerFlags )
CheckBLACSCompilerFlags()

set(prefix ${CMAKE_INSTALL_PREFIX})
set(libdir ${CMAKE_INSTALL_PREFIX}/lib${LIB_SUFFIX})
set(PKG_CONFIG_DIR ${libdir}/pkgconfig)

# --------------------------------------------------
# ScaLAPACK needs BLAS and LAPACK
message(STATUS "Checking BLAS and LAPACK libraries")

include(CheckFortranFunctionExists)
set(CMAKE_REQUIRED_LIBRARIES lapack)
CHECK_FORTRAN_FUNCTION_EXISTS("dgesv" LAPACK_FOUND)
unset( CMAKE_REQUIRED_LIBRARIES )
if (NOT LAPACK_FOUND)
    message(ERROR "LAPACK routine dgesv not found. Check your LAPACK version")
endif()

# --------------------------------------------------
# Subdirectories that need to be processed
   
macro(append_subdir_files variable dirname)
    get_directory_property(holder DIRECTORY ${dirname} DEFINITION ${variable})
    foreach(depfile ${holder})
        if (EXISTS "${CMAKE_CURRENT_LIST_DIR}/${dirname}/${depfile}")
            list(APPEND ${variable} "${CMAKE_CURRENT_LIST_DIR}/${dirname}/${depfile}")
        elseif (EXISTS "${CMAKE_CURRENT_BINARY_DIR}/${dirname}/${depfile}")
            list(APPEND ${variable} "${CMAKE_CURRENT_BINARY_DIR}/${dirname}/${depfile}")
        else ()
            message(STATUS "Trying to add a non-existent file. '${depfile}'")
        endif()
    endforeach()
endmacro()

#
# BLACS
#
add_subdirectory(BLACS)
append_subdir_files(blacs "BLACS/SRC")

#
# TOOLS
#
add_subdirectory(TOOLS)
append_subdir_files(tools TOOLS)
append_subdir_files(tools-C TOOLS)
append_subdir_files(extra_lapack "TOOLS/LAPACK")

#
# PBLAS
#
add_subdirectory(PBLAS)
append_subdir_files(pblas "PBLAS/SRC")
append_subdir_files(pblas-F "PBLAS/SRC")
append_subdir_files(pbblas "PBLAS/SRC/PBBLAS")
append_subdir_files(ptzblas "PBLAS/SRC/PTZBLAS")
append_subdir_files(ptools "PBLAS/SRC/PTOOLS")

#
# REDIST
#
add_subdirectory(REDIST)
append_subdir_files(redist "REDIST/SRC")

#
# SRC
#
add_subdirectory(SRC)
append_subdir_files(src "SRC")
append_subdir_files(src-C "SRC")

# Need to separate Fortran and C Code on MSVC windows (projects can only have single source types)
if (MSVC)
    set(SCALAPACK_EXPORT_LIBRARIES scalapack_Fortran scalapack)
    add_library(scalapack_Fortran ${pblas-F} ${pbblas} ${ptzblas} ${tools} ${src} ${extra_lapack} )
    target_link_libraries(scalapack_Fortran PUBLIC lapack blas MPI::MPI_Fortran)
    target_compile_definitions(scalapack_Fortran PUBLIC ${FORTRAN_MANGLING})
    set_target_properties(scalapack_Fortran PROPERTIES OUTPUT_NAME scalapack_Fortran-${SCALAPACK_VERSION})
    add_library(scalapack ${blacs} ${tools-C} ${pblas} ${ptools} ${redist} ${src-C})
    target_link_libraries(scalapack PUBLIC scalapack_Fortran MPI::MPI_C )
    # set_target_properties(scalapack PROPERTIES LINKER_LANGUAGE C)
else()
    set(SCALAPACK_EXPORT_LIBRARIES scalapack)
    add_library(scalapack ${blacs} ${tools} ${tools-C} ${extra_lapack} ${pblas} ${pblas-F} ${ptzblas} ${ptools} ${pbblas} ${redist} ${src} ${src-C})
    target_compile_definitions(scalapack PUBLIC ${FORTRAN_MANGLING})
    target_link_libraries(scalapack PUBLIC lapack blas MPI::MPI_C MPI::MPI_Fortran)
endif()
set_target_properties(scalapack PROPERTIES OUTPUT_NAME scalapack-${SCALAPACK_VERSION})
if (MSVC)
    set_target_properties(scalapack PROPERTIES LINKER_LANGUAGE C)
endif ()

message(STATUS "Using Fortran mangling: ${FORTRAN_MANGLING}")

install(TARGETS ${SCALAPACK_EXPORT_LIBRARIES} EXPORT scalapack-config
    DESTINATION lib)
install(EXPORT scalapack-config DESTINATION ${PACKAGE_CONFIG_DIR})
include(CMakePackageConfigHelpers)
WRITE_BASIC_PACKAGE_VERSION_FILE(${CMAKE_CURRENT_BINARY_DIR}/scalapack-config-version.cmake COMPATIBILITY AnyNewerVersion)
file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/scalapack-config-dependencies.cmake 
    "include(CMakeFindDependencyMacro)\r\n"
    "set(${PROJECT_NAME}_IMPORT_PREFIX \${_IMPORT_PREFIX})\r\n"
)
file(APPEND ${CMAKE_CURRENT_BINARY_DIR}/scalapack-config-dependencies.cmake "set(LAPACK_FIND_SYSTEM ${LAPACK_FIND_SYSTEM})\r\nfind_dependency(LAPACK ${LAPACK_VERSION})\r\n")
file(APPEND ${CMAKE_CURRENT_BINARY_DIR}/scalapack-config-dependencies.cmake "find_dependency(MPI)\r\n")
file(APPEND ${CMAKE_CURRENT_BINARY_DIR}/scalapack-config-dependencies.cmake "set(_IMPORT_PREFIX \${${PROJECT_NAME}_IMPORT_PREFIX})")
install(FILES 
    ${CMAKE_CURRENT_BINARY_DIR}/scalapack-config-version.cmake
    ${CMAKE_CURRENT_BINARY_DIR}/scalapack-config-dependencies.cmake
    DESTINATION ${PACKAGE_CONFIG_DIR})

#
# TESTING
#
if (BUILD_TESTS)
    add_subdirectory(TESTING)
endif()
