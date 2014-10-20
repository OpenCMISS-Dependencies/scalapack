cmake_minimum_required(VERSION 2.8)
project(SCALAPACK C Fortran)

SET(CMAKE_PREFIX_PATH ${OPENCMISS_DEPENDENCIES_CONFIGS_DIR} ${CMAKE_PREFIX_PATH})
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${OPENCMISS_DEPENDENCIES_LIBRARIES})
FIND_PACKAGE(BLAS CONFIG REQUIRED)
FIND_PACKAGE(LAPACK CONFIG REQUIRED)

# Configure the warning and code coverage suppression file
configure_file( 
  "${SCALAPACK_SOURCE_DIR}/CMAKE/CTestCustom.cmake.in"
  "${SCALAPACK_BINARY_DIR}/CTestCustom.cmake"
  COPYONLY
)

# Add the CMake directory for custon CMake modules
set(CMAKE_MODULE_PATH "${SCALAPACK_SOURCE_DIR}/CMAKE" ${CMAKE_MODULE_PATH})

if (UNIX)
   if ( "${CMAKE_Fortran_COMPILER}" MATCHES "ifort" )
  set( CMAKE_Fortran_FLAGS "${CMAKE_Fortran_FLAGS} -fltconsistency -fp_port" )
   endif ()
endif ()

#INCLUDE_DIRECTORIES(${MPI_C_INCLUDE_PATH} ${MPI_Fortran_INCLUDE_PATH})
find_package(MPI REQUIRED)
if (MPI_FOUND)
   message(STATUS "Found MPI_LIBRARY : ${MPI_FOUND} ")
   INCLUDE_DIRECTORIES(${MPI_INCLUDE_PATH})

   find_program(MPI_C_COMPILER
      NAMES mpicc
      HINTS "${MPI_BASE_DIR}"
      PATH_SUFFIXES bin
      DOC "MPI C compiler.")
   MARK_AS_ADVANCED(MPI_C_COMPILER)
   if ("${MPI_C_COMPILER}" STREQUAL "MPI_C_COMPILER-NOTFOUND")
      message(ERROR "--> MPI C Compiler NOT FOUND (please set MPI_BASE_DIR accordingly")
   else()
      message(STATUS "--> MPI C Compiler : ${MPI_C_COMPILER}")
      SET(CMAKE_C_COMPILER "${MPI_C_COMPILER}")
      message(STATUS "--> C Compiler : ${CMAKE_C_COMPILER}")
   endif()
   find_program(MPI_Fortran_COMPILER
      NAMES mpif77
      HINTS "${MPI_BASE_DIR}"
      PATH_SUFFIXES bin
      DOC "MPI Fortran compiler.")
   MARK_AS_ADVANCED(MPI_Fortran_COMPILER)

   if ("${MPI_Fortran_COMPILER}" STREQUAL "MPI_Fortran_COMPILER-NOTFOUND")
      message(ERROR "--> MPI Fortran Compiler NOT FOUND (please set MPI_BASE_DIR accordingly")
   else()
      message(STATUS "--> MPI Fortran Compiler : ${MPI_Fortran_COMPILER}")
      SET(Fortran_COMPILER "${CMAKE_Fortran_COMPILER}")
      SET(CMAKE_Fortran_COMPILER "${MPI_Fortran_COMPILER}")
      message(STATUS "--> Fortran Compiler : ${CMAKE_Fortran_COMPILER}")
   endif()   
else()
   message(STATUS "Found MPI_LIBRARY : ${MPI_FOUND} ")
   set(MPI_BASE_DIR ${MPI_BASE_DIR} CACHE PATH "MPI Path")
   UNSET(MPIEXEC CACHE)
   UNSET(MPIEXEC_POSTFLAGS CACHE)
   UNSET(MPIEXEC_PREFLAGS CACHE)
   UNSET(MPIEXEC_MAX_NUMPROCS CACHE)
   UNSET(MPIEXEC_NUMPROC_FLAG CACHE)
   UNSET(MPI_COMPILE_FLAGS CACHE)
   UNSET(MPI_LINK_FLAGS CACHE)
   UNSET(MPI_INCLUDE_PATH CACHE)
   message(FATAL_ERROR "--> MPI Library NOT FOUND -- please set MPI_BASE_DIR accordingly --")
endif()

if (UNIX)
   if ( "${CMAKE_Fortran_COMPILER}" MATCHES "ifort" )
  set( CMAKE_Fortran_FLAGS "${CMAKE_Fortran_FLAGS} -fltconsistency -fp_port" )
   endif ()
endif ()

#macro(SCALAPACK_install_library lib)
#  install(TARGETS ${lib} EXPORT scalapack-targets
#    ARCHIVE DESTINATION lib${LIB_SUFFIX}
#    LIBRARY DESTINATION lib${LIB_SUFFIX}
#    RUNTIME DESTINATION Testing
#  )
#endmacro()

# --------------------------------------------------
# Testing
SET(DART_TESTING_TIMEOUT 600)

enable_testing()
include(CTest)
enable_testing()
# --------------------------------------------------

# Organize output files.  On Windows this also keeps .dll files next
# to the .exe files that need them, making tests easy to run.
#set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${SCALAPACK_BINARY_DIR}/TESTING)
#set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${SCALAPACK_BINARY_DIR}/lib)
#set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${SCALAPACK_BINARY_DIR}/lib)

# --------------------------------------------------
# Check for any necessary platform specific compiler flags
include( CheckBLACSCompilerFlags )
CheckBLACSCompilerFlags()

set(prefix ${CMAKE_INSTALL_PREFIX})
set(libdir ${CMAKE_INSTALL_PREFIX}/lib${LIB_SUFFIX})
set(PKG_CONFIG_DIR ${libdir}/pkgconfig)

# --------------------------------------------------
# BLACS Internal variables
#
#   Fortran Mangling, MPI Tests and BLACS settings
#
include(FortranMangling)
COMPILE(install_COMPILED)

FORTRAN_MANGLING(CDEFS)
#MESSAGE(STATUS "Setting CDEFS = ${CDEFS}")
#set(CDEFS ${CDEFS} CACHE STRING "Fortran Mangling" FORCE)
MESSAGE(STATUS "=========")

# --------------------------------------------------
# Compiler Flags
ADD_DEFINITIONS( "-D${CDEFS}")

# --------------------------------------------------
# ScaLAPACK needs BLAS and LAPACK
#option(USE_OPTIMIZED_LAPACK_BLAS "Whether or not to search for optimized LAPACK and BLAS libraries on your machine (if not found, Reference LAPACK and BLAS will be downloaded and installed)" ON)

message(STATUS "ScaLAPACK BLAS library: ${BLAS_LIBRARIES}")
message(STATUS "ScaLAPACK LAPACK library: ${LAPACK_LIBRARIES}")

MESSAGE(STATUS "=========")

# --------------------------------------------------
# By default static library
OPTION(BUILD_STATIC_LIBS "Build static libraries" ON )

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
#add_subdirectory(REDIST)
#append_subdir_files(redist "REDIST/SRC")

#
# SRC
#
add_subdirectory(SRC)
append_subdir_files(src "SRC")
append_subdir_files(src-C "SRC")

#message(STATUS "Linking ScaLAPACK to lapack/blas libs: ${LAPACK_LIBRARIES} ${BLAS_LIBRARIES}")
if (UNIX)
   add_library(scalapack ${blacs} ${tools} ${tools-C} ${extra_lapack} ${pblas} ${pblas-F} ${ptzblas} ${ptools} ${pbblas} ${redist} ${src} ${src-C})
   target_link_libraries( scalapack lapack blas)
   #scalapack_install_library(scalapack)
else (UNIX) # Need to separate Fortran and C Code
   OPTION(BUILD_SHARED_LIBS "Build shared libraries" ON )
   add_library(scalapack ${blacs} ${tools-C} ${pblas} ${ptools} ${redist} ${src-C})
   target_link_libraries( scalapack lapack blas)
   add_library(scalapack-F ${pblas-F} ${pbblas} ${ptzblas} ${tools} ${src} ${extra_lapack} )
   target_link_libraries( scalapack-F lapack blas)
   #scalapack_install_library(scalapack)
   #scalapack_install_library(scalapack-F)
endif (UNIX)
#add_subdirectory(TESTING)

# --------------------------------------------------
export(TARGETS scalapack FILE ${OPENCMISS_DEPENDENCIES_CONFIGS_DIR}/scalapack-config.cmake)

#if( NOT LAPACK_FOUND )
# install(FILES
#  ${BLAS_LIBRARIES} ${LAPACK_LIBRARIES}
#  DESTINATION lib
#   )
#endif( NOT LAPACK_FOUND )

#configure_file(${SCALAPACK_SOURCE_DIR}/CMAKE/scalapack-config-version.cmake.in
#  ${SCALAPACK_BINARY_DIR}/scalapack-config-version.cmake @ONLY)
#configure_file(${SCALAPACK_SOURCE_DIR}/CMAKE/scalapack-config-build.cmake.in
#  ${SCALAPACK_BINARY_DIR}/scalapack-config.cmake @ONLY)

#configure_file(${CMAKE_CURRENT_SOURCE_DIR}/scalapack.pc.in ${CMAKE_CURRENT_BINARY_DIR}/scalapack.pc)
# install(FILES
#  ${CMAKE_CURRENT_BINARY_DIR}/scalapack.pc
#  DESTINATION ${PKG_CONFIG_DIR}
#   )

#configure_file(${SCALAPACK_SOURCE_DIR}/CMAKE/scalapack-config-install.cmake.in
#  ${SCALAPACK_BINARY_DIR}/CMakeFiles/scalapack-config.cmake @ONLY)
#install(FILES
#  ${SCALAPACK_BINARY_DIR}/CMakeFiles/scalapack-config.cmake
#  ${SCALAPACK_BINARY_DIR}/scalapack-config-version.cmake
#  DESTINATION lib/cmake/scalapack-${SCALAPACK_VERSION}
#  )

#install(EXPORT scalapack-targets
#  DESTINATION lib/cmake/scalapack-${SCALAPACK_VERSION})

