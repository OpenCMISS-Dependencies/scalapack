file(COPY ../SRC/PTOOLS/PB_Cwarn.c DESTINATION ${CMAKE_CURRENT_SOURCE_DIR})
file(COPY ../SRC/PTOOLS/PB_Cabort.c DESTINATION ${CMAKE_CURRENT_SOURCE_DIR})

set (PblasErrorHandler  PB_Cwarn.c PB_Cabort.c)
set (pbtcom pblastst.f ${PblasErrorHandler})

set_property(
   SOURCE ${PblasErrorHandler}
   APPEND PROPERTY COMPILE_DEFINITIONS TestingPblas 
   )

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${SCALAPACK_BINARY_DIR}/PBLAS/TESTING)

foreach(prec s d c z)
    if(BUILD_PRECISION MATCHES [${prec}])
        set (common p${prec}blastst.f ${prec}lamch.f ${pbtcom})
        string(TOUPPER ${prec} PREC)
        file(COPY P${PREC}BLAS1TST.dat DESTINATION ${CMAKE_RUNTIME_OUTPUT_DIRECTORY})
        file(COPY P${PREC}BLAS2TST.dat DESTINATION ${CMAKE_RUNTIME_OUTPUT_DIRECTORY})
        file(COPY P${PREC}BLAS3TST.dat DESTINATION ${CMAKE_RUNTIME_OUTPUT_DIRECTORY})
        add_executable(${prec}pb1tst p${prec}blas1tst.f ${common})
        add_executable(${prec}pb2tst p${prec}blas2tst.f ${common})
        add_executable(${prec}pb3tst p${prec}blas3tst.f ${common})
        target_link_libraries(${prec}pb1tst scalapack lapack blas ${MPI_Fortran_LIBRARIES})
        target_link_libraries(${prec}pb2tst scalapack lapack blas ${MPI_Fortran_LIBRARIES})
        target_link_libraries(${prec}pb3tst scalapack lapack blas ${MPI_Fortran_LIBRARIES})
        add_test(${prec}pb1tst ${MPIEXEC} ${MPIEXEC_NUMPROC_FLAG} 4 ./${prec}pb1tst)
        add_test(${prec}pb2tst ${MPIEXEC} ${MPIEXEC_NUMPROC_FLAG} 4 ./${prec}pb2tst)
        add_test(${prec}pb3tst ${MPIEXEC} ${MPIEXEC_NUMPROC_FLAG} 4 ./${prec}pb3tst)
    endif()
endforeach()