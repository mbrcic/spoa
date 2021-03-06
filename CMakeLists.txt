cmake_minimum_required(VERSION 3.2)
project(spoa LANGUAGES CXX VERSION 3.0.0)

include(GNUInstallDirs)

set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/lib)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/bin)

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra -pedantic")
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

option(spoa_build_executable "Build spoa standalone tool" OFF)
option(spoa_build_tests "Build spoa unit tests" OFF)
option(spoa_optimize_for_native "Build spoa with -march=native" ON)
option(spoa_optimize_for_portability "Build spoa with -msse4.1" OFF)
option(spoa_use_simde "Use SIMDe library for porting vectorized code" OFF)
option(spoa_use_simde_nonvec "Use SIMDe library for nonvectorized code" OFF)
option(spoa_use_simde_openmp "Use SIMDe support for OpenMP SIMD" OFF)

if (spoa_optimize_for_portability)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -msse4.1")
elseif (spoa_optimize_for_native)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -march=native")
endif()

if (spoa_use_simde OR spoa_use_simde_nonvec OR spoa_use_simde_openmp)
    add_definitions(-DUSE_SIMDE -DSIMDE_ENABLE_NATIVE_ALIASES)
    if (spoa_use_simde_nonvec)
        add_definitions(-DSIMDE_NO_NATIVE)
    endif()
    if (spoa_use_simde_openmp)
        add_definitions(-DSIMDE_ENABLE_OPENMP)
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fopenmp-simd")
    endif()
endif()

# build SPOA as a static library by default
set(BUILD_SHARED_LIBS OFF CACHE BOOL "Build all libraries as shared")

add_library(spoa
    src/alignment_engine.cpp
    src/graph.cpp
    src/simd_alignment_engine.cpp
    src/sisd_alignment_engine.cpp)

target_include_directories(spoa PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/vendor/simde>)

set_target_properties(spoa
    PROPERTIES
    VERSION ${spoa_VERSION}
    SOVERSION ${spoa_VERSION})

install(TARGETS spoa DESTINATION ${CMAKE_INSTALL_LIBDIR})
install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/include/spoa DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

# configure and install pkg-config file
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/spoa.pc.in ${CMAKE_CURRENT_BINARY_DIR}/spoa-1.pc @ONLY)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/spoa-1.pc DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)

if (spoa_build_executable)
    add_executable(spoa_bin
        src/sequence.cpp
        src/main.cpp)

    if (NOT TARGET bioparser)
        add_subdirectory(vendor/bioparser EXCLUDE_FROM_ALL)
    endif()

    target_link_libraries(spoa_bin spoa bioparser)
    set_target_properties(spoa_bin PROPERTIES OUTPUT_NAME spoa)

    install(TARGETS spoa_bin DESTINATION ${CMAKE_INSTALL_BINDIR})
endif()

if (spoa_build_tests)
    set(spoa_test_data_path ${PROJECT_SOURCE_DIR}/test/data/)
    configure_file(${PROJECT_SOURCE_DIR}/test/spoa_test_config.h.in
        ${PROJECT_BINARY_DIR}/config/spoa_test_config.h)
    include_directories(${PROJECT_BINARY_DIR}/config)
    include_directories(${PROJECT_SOURCE_DIR}/src)

    add_executable(spoa_test
        src/sequence.cpp
        test/spoa_test.cpp)

    if (NOT TARGET bioparser)
        add_subdirectory(vendor/bioparser EXCLUDE_FROM_ALL)
    endif()
    if (NOT TARGET gtest_main)
        add_subdirectory(vendor/googletest/googletest EXCLUDE_FROM_ALL)
    endif()

    target_link_libraries(spoa_test spoa bioparser gtest_main)
endif()
