cmake_minimum_required(VERSION 3.0)
project(SCALAPACK VERSION 2.8 LANGUAGES C Fortran)

option(BUILD_TESTS "${PROJECT_NAME} - Build tests" ON)
option(PACKAGE_CONFIG_DIR "Directory for package config files (relative to CMAKE_INSTALL_PREFIX)" "lib/cmake")
option(FORTRAN_MANGLING "${PROJECT_NAME} - Fortran mangling scheme" Add_)
set(BUILD_PRECISION sdcz CACHE STRING "${PROJECT_NAME} - Build precisions (sdcz)")

# Always build single precision (used by some tests [e.g. PETSc] and examples)
set(BUILD_PRECISION "s${BUILD_PRECISION}")

find_package(MPI REQUIRED)
find_package(BLAS ${BLAS_VERSION} REQUIRED)
find_package(LAPACK ${LAPACK_VERSION} REQUIRED)

# Configure the warning and code coverage suppression file
configure_file( 
  "${SCALAPACK_SOURCE_DIR}/CMAKE/CTestCustom.cmake.in"
  "${SCALAPACK_BINARY_DIR}/CTestCustom.cmake"
  COPYONLY
)

# Add the CMake directory for custon CMake modules
set(CMAKE_MODULE_PATH "${SCALAPACK_SOURCE_DIR}/CMAKE" ${CMAKE_MODULE_PATH})

#
#   MPI
#
if (MPI_FOUND)
   INCLUDE_DIRECTORIES(${MPI_INCLUDE_PATH})
   if (MPI_C_COMPILER)
      SET(CMAKE_C_COMPILER "${MPI_C_COMPILER}")
      message(STATUS "--> C Compiler : ${CMAKE_C_COMPILER}")
   endif()
   if (MPI_Fortran_COMPILER)
      SET(Fortran_COMPILER "${CMAKE_Fortran_COMPILER}")
      SET(CMAKE_Fortran_COMPILER "${MPI_Fortran_COMPILER}")
      message(STATUS "--> Fortran Compiler : ${CMAKE_Fortran_COMPILER}")
   endif()
endif()

if (UNIX)
   if ( "${CMAKE_Fortran_COMPILER}" MATCHES "ifort" )
      set( CMAKE_Fortran_FLAGS "${CMAKE_Fortran_FLAGS} -fltconsistency -fp_port" )
   endif ()
endif ()

# --------------------------------------------------
# Testing
if (BUILD_TESTS)
    SET(DART_TESTING_TIMEOUT 600)
    enable_testing()
    include(CTest)
    enable_testing()
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
# BLACS Internal variables
#
#   Fortran Mangling, MPI Tests and BLACS settings
#

# --------------------------------------------------
# Compiler Flags
ADD_DEFINITIONS("-D${FORTRAN_MANGLING}")
#ADD_DEFINITIONS("-DNoChange")

# --------------------------------------------------
# ScaLAPACK needs BLAS and LAPACK
message(STATUS "CHECKING BLAS AND LAPACK LIBRARIES")

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
      list(APPEND ${variable} "${dirname}/${depfile}")
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

if (NOT MSVC)
   add_library(scalapack ${blacs} ${tools} ${tools-C} ${extra_lapack} ${pblas} ${pblas-F} ${ptzblas} ${ptools} ${pbblas} ${redist} ${src} ${src-C})
   target_link_libraries(scalapack PUBLIC lapack blas ${MPI_C_LIBRARIES} ${MPI_Fortran_LIBRARIES})
else() # Need to separate Fortran and C Code on MSVC windows (projects can only have single source types)
   add_library(scalapack-C OBJECT ${blacs} ${tools-C} ${pblas} ${ptools} ${redist} ${src-C})
   #target_link_libraries( scalapack-C PUBLIC lapack blas ${MPI_C_LIBRARIES} ${MPI_Fortran_LIBRARIES})
   add_library(scalapack-F OBJECT ${pblas-F} ${pbblas} ${ptzblas} ${tools} ${src} ${extra_lapack} )
   #target_link_libraries( scalapack-F PUBLIC lapack blas ${MPI_C_LIBRARIES} ${MPI_Fortran_LIBRARIES})
   add_library(scalapack $<TARGET_OBJECTS:scalapack-C> $<TARGET_OBJECTS:scalapack-F>)
   target_link_libraries(scalapack PUBLIC lapack blas ${MPI_C_LIBRARIES} ${MPI_Fortran_LIBRARIES})
endif()

set_target_properties(scalapack PROPERTIES OUTPUT_NAME scalapack-${SCALAPACK_VERSION})

install(TARGETS scalapack EXPORT scalapack-config
    DESTINATION lib)
install(EXPORT scalapack-config DESTINATION ${PACKAGE_CONFIG_DIR})
include(CMakePackageConfigHelpers)
WRITE_BASIC_PACKAGE_VERSION_FILE(${CMAKE_CURRENT_BINARY_DIR}/scalapack-config-version.cmake COMPATIBILITY AnyNewerVersion)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/scalapack-config-version.cmake
    DESTINATION lib/cmake)

#
# TESTING
#
if (BUILD_TESTS)
    add_subdirectory(TESTING)
endif()