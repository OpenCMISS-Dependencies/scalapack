set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${SCALAPACK_BINARY_DIR}/TESTING)

foreach(prec s d c z)
    if(BUILD_PRECISION MATCHES [${prec}])
        set (matgen p${prec}matgen.f pmatgeninc.f)
        add_executable(x${prec}brd p${prec}brddriver.f p${prec}brdinfo.f p${prec}gebdrv.f p${prec}lafchk.f ${matgen})
        add_executable(x${prec}hrd p${prec}hrddriver.f p${prec}hrdinfo.f p${prec}gehdrv.f p${prec}lafchk.f ${matgen})
        
        set(SRC p${prec}trddriver.f p${prec}ttrdtester.f p${prec}latran.f p${prec}trdinfo.f p${prec}lafchk.f xpjlaenv.f ${matgen})
        if (NOT ${prec} MATCHES [cz])
            list(APPEND SRC p${prec}sytdrv.f)
        else()
            list(APPEND SRC p${prec}hetdrv.f)
        endif()
        add_executable(x${prec}trd ${SRC})

        set(SRC p${prec}septst.f p${prec}sepsubtst.f p${prec}sepchk.f p${prec}sepqtq.f p${prec}lagsy.f 
            p${prec}latms.f p${prec}lasizesep.f  p${prec}sepdriver.f
            p${prec}sepreq.f p${prec}sdpsubtst.f ${matgen})
        if (NOT ${prec} MATCHES [cz])
            list(APPEND SRC p${prec}lasizesyevx.f p${prec}lasizesyev.f 
                p${prec}sepinfo.f p${prec}sqpsubtst.f
                p${prec}lasizesqp.f)
        else()
            list(APPEND SRC p${prec}lasizeheevx.f p${prec}sepreq.f 
            pdsepinfo.f pssepinfo.f p${prec}septst.f p${prec}sepsubtst.f p${prec}sdpsubtst.f
            p${prec}sepchk.f p${prec}sepqtq.f)
        endif()
        add_executable(x${prec}sep ${SRC})
        
        set(SRC p${prec}gseptst.f p${prec}gsepsubtst.f p${prec}gsepchk.f p${prec}lagsy.f
            p${prec}latms.f p${prec}lasizegsep.f p${prec}lasizesep.f
            p${prec}gsepdriver.f p${prec}gsepreq.f ${matgen})
        if (NOT ${prec} MATCHES [cz])
            list(APPEND SRC p${prec}lasizesyevx.f p${prec}sepinfo.f)
        else()
            list(APPEND SRC p${prec}lasizeheevx.f p${prec}sepreq.f 
            pdsepinfo.f pssepinfo.f p${prec}septst.f p${prec}sepsubtst.f p${prec}sdpsubtst.f
            p${prec}sepchk.f p${prec}sepqtq.f)
        endif() 
        add_executable(x${prec}gsep ${SRC})
        add_executable(x${prec}nep p${prec}nepdriver.f p${prec}nepinfo.f p${prec}nepfchk.f ${matgen})
        
        set(SRC p${prec}lasizesepr.f p${prec}seprdriver.f p${prec}seprreq.f p${prec}seprsubtst.f
            p${prec}sepchk.f p${prec}sepqtq.f p${prec}latms.f p${prec}seprtst.f  
            p${prec}lagsy.f p${prec}lasizesep.f ${matgen})
        if (NOT ${prec} MATCHES [cz])
            list(APPEND SRC p${prec}lasizesyevr.f p${prec}sepinfo.f)
        else()
            list(APPEND SRC p${prec}lasizeheevr.f pssepinfo.f p${prec}seprreq.f pdsepinfo.f)
        endif()
        add_executable(x${prec}syevr ${SRC})

        target_link_libraries(x${prec}brd scalapack)
        target_link_libraries(x${prec}hrd scalapack)
        target_link_libraries(x${prec}trd scalapack)
        target_link_libraries(x${prec}sep scalapack)
        target_link_libraries(x${prec}gsep scalapack)
        target_link_libraries(x${prec}nep scalapack)
        target_link_libraries(x${prec}syevr scalapack)
    endif()
endforeach()
if(BUILD_PRECISION MATCHES [s])
    add_executable(xssvd pssvddriver.f pslagge.f pssvdchk.f pssvdcmp.f pssvdtst.f psmatgen.f pmatgeninc.f)
    target_link_libraries(xssvd scalapack)
    add_executable(xshseqr pshseqrdriver.f psmatgen2.f pcmatgen.f pmatgeninc.f)
    target_link_libraries(xshseqr scalapack)
endif()
if(BUILD_PRECISION MATCHES [d])
    add_executable(xdsvd pdsvddriver.f pdlagge.f pdsvdchk.f pdsvdcmp.f pdsvdtst.f pdmatgen.f pmatgeninc.f)
    target_link_libraries(xdsvd scalapack)
    add_executable(xdhseqr pdhseqrdriver.f pdmatgen2.f pcmatgen.f pmatgeninc.f)
    target_link_libraries(xdhseqr scalapack)
endif()
if(BUILD_PRECISION MATCHES [c])
    add_executable(xcevc pcevcdriver.f pcevcinfo.f pcget22.f pcmatgen.f pmatgeninc.f)
    target_link_libraries(xcevc scalapack)
endif()
if(BUILD_PRECISION MATCHES [z])
    add_executable(xzevc pzevcdriver.f pzevcinfo.f pzget22.f pzmatgen.f pmatgeninc.f)
    target_link_libraries(xzevc scalapack)
endif()
