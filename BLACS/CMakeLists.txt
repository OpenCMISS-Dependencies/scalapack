add_subdirectory(SRC)
if(BUILD_TESTS AND NOT MSVC)
  add_subdirectory(TESTING)
endif()
