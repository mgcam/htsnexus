cmake_minimum_required(VERSION 2.8)
include(ExternalProject)

IF(NOT CMAKE_BUILD_TYPE)
  SET(CMAKE_BUILD_TYPE Release CACHE STRING
      "Choose the type of build, options are: None Debug Release RelWithDebInfo MinSizeRel."
      FORCE)
ENDIF(NOT CMAKE_BUILD_TYPE)

# Make PROJECT_SOURCE_DIR, PROJECT_BINARY_DIR, and PROJECT_NAME available.
project(htsnexus)

# After build, double-check external/src/htslib-stamp/htslib-build-out.log
# to make sure CFLAGS were set as expected
ExternalProject_Add(htslib
    URL https://github.com/samtools/htslib/archive/1.3.zip
    PREFIX ${CMAKE_CURRENT_BINARY_DIR}/external
    CONFIGURE_COMMAND ""
    BUILD_IN_SOURCE 1
    BUILD_COMMAND make -j4
    INSTALL_COMMAND ""
    LOG_DOWNLOAD ON
    LOG_BUILD ON
  )
ExternalProject_Get_Property(htslib source_dir)
include_directories(${source_dir})
ExternalProject_Get_Property(htslib binary_dir)
add_library(libhts STATIC IMPORTED)
set(LIBHTS_A ${binary_dir}/libhts.a)
set_property(TARGET libhts PROPERTY IMPORTED_LOCATION ${LIBHTS_A})

# Also build samtools executable for use in integration tests
ExternalProject_Add(samtools
    URL https://github.com/samtools/samtools/archive/1.3.zip
    PREFIX ${CMAKE_CURRENT_BINARY_DIR}/external
    CONFIGURE_COMMAND ""
    BUILD_IN_SOURCE 1
    BUILD_COMMAND make -j4
    INSTALL_COMMAND ""
    LOG_DOWNLOAD ON
    LOG_BUILD ON
  )
add_dependencies(samtools htslib)

execute_process(COMMAND git describe --tags --long --dirty --always
                OUTPUT_VARIABLE GIT_REVISION OUTPUT_STRIP_TRAILING_WHITESPACE)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DGIT_REVISION=\"\\\"${GIT_REVISION}\\\"\" -pthread -std=c++11 -Wall")
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -static-libgcc -static-libstdc++")

################################
# Normal Libraries & Executables
################################
include_directories(src)

add_executable(htsnexus_index_bam src/htsnexus_index_bam.cc src/htsnexus_index_util.cc)
add_dependencies(htsnexus_index_bam htslib)
target_link_libraries(htsnexus_index_bam libhts sqlite3 z)

add_executable(htsnexus_index_cram src/htsnexus_index_cram.cc src/htsnexus_index_util.cc)
add_dependencies(htsnexus_index_cram htslib samtools)
target_link_libraries(htsnexus_index_cram libhts sqlite3 z)

install(TARGETS htsnexus_index_bam htsnexus_index_cram DESTINATION bin)

################################
# Testing
################################

#add_test(unit_tests unit_tests)
