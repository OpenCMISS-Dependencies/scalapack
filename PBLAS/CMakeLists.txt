add_subdirectory(SRC)
if (BUILD_TESTS)
    add_subdirectory(TESTING)
    add_subdirectory(TIMING)
endif()

