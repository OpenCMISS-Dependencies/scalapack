set(FTestObj  
   blacstest.f btprim.f tools.f)

add_executable(xFbtest ${FTestObj})
target_include_directories(xFbtest PUBLIC ${MPI_Fortran_INCLUDE_PATH})
target_link_libraries(xFbtest scalapack ${MPI_Fortran_LIBRARIES})

add_library(xCbtest_c Cbt.c)
target_compile_definitions(xCbtest_c PUBLIC BTCINTFACE)
target_include_directories(xCbtest_c PUBLIC ${MPI_Fortran_INCLUDE_PATH})
target_link_libraries(xCbtest_c PUBLIC scalapack ${MPI_C_LIBRARIES})

add_executable(xCbtest ${FTestObj})
target_include_directories(xCbtest PUBLIC ${MPI_Fortran_INCLUDE_PATH})
target_link_libraries(xCbtest xCbtest_c)
target_link_libraries(xCbtest ${MPI_Fortran_LIBRARIES})

file(COPY bsbr.dat DESTINATION ${SCALAPACK_BINARY_DIR}/BLACS/TESTING)
file(COPY bt.dat DESTINATION ${SCALAPACK_BINARY_DIR}/BLACS/TESTING)
file(COPY comb.dat DESTINATION ${SCALAPACK_BINARY_DIR}/BLACS/TESTING)
file(COPY sdrv.dat DESTINATION ${SCALAPACK_BINARY_DIR}/BLACS/TESTING)

# We could run the BLACS TESTING the following way
# But BLACS TESTING are TESTING anormal exit so even if they pass,
# CTest will determine they fail
#add_test(xFbtest0 ${MPIEXEC} ${MPIEXEC_NUMPROC_FLAG} 4 ./xFbtest)
#add_test(xCbtest0 ${MPIEXEC} ${MPIEXEC_NUMPROC_FLAG} 4 ./xCbtest)

add_test(xCbtest
     ${CMAKE_COMMAND}
    -DMPIEXEC=${MPIEXEC}
    -DMPIEXEC_NUMPROC_FLAG=${MPIEXEC_NUMPROC_FLAG}
    -DTEST_PROG=xCbtest
    -DOUTPUTDIR=${SCALAPACK_BINARY_DIR}/BLACS/TESTING
    -DRUNTIMEDIR=${CMAKE_RUNTIME_OUTPUT_DIRECTORY}
    -DSOURCEDIR=${CMAKE_CURRENT_SOURCE_DIR}
    -P ${CMAKE_CURRENT_SOURCE_DIR}/runtest.cmake
    )

add_test(xFbtest
     ${CMAKE_COMMAND}
    -DMPIEXEC=${MPIEXEC}
    -DMPIEXEC_NUMPROC_FLAG=${MPIEXEC_NUMPROC_FLAG}
    -DTEST_PROG=xFbtest
    -DOUTPUTDIR=${SCALAPACK_BINARY_DIR}/BLACS/TESTING
    -DRUNTIMEDIR=${CMAKE_RUNTIME_OUTPUT_DIRECTORY}
    -DSOURCEDIR=${CMAKE_CURRENT_SOURCE_DIR}
    -P ${CMAKE_CURRENT_SOURCE_DIR}/runtest.cmake
    )
