cmake_minimum_required(VERSION 3.14)

project(nnue_gen VERSION 1.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release)
endif()

if(WIN32)
    add_definitions(-D_WIN32)
elseif(UNIX AND NOT APPLE)
    add_definitions(-DLINUX)
elseif(APPLE)
    add_definitions(-DMACOS)
endif()

if(MSVC)
    add_compile_options(/W4)
else()
    add_compile_options(-Wall -Wextra -pedantic)
endif()

include(FetchContent)

# moku core (fetched); builds only moku_core as a subproject
FetchContent_Declare(
    moku
    GIT_REPOSITORY https://github.com/gn0rrbitberrenbur/moku.git
    GIT_TAG        v0.0.5
)
FetchContent_MakeAvailable(moku)

# googletest (moku only fetches it when top-level)
FetchContent_Declare(
    googletest
    GIT_REPOSITORY https://github.com/google/googletest.git
    GIT_TAG        v1.14.0
)
set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(googletest)

find_package(Threads REQUIRED)

set(NNUE_MOKU_INCLUDES
    ${CMAKE_CURRENT_SOURCE_DIR}/include
    ${moku_SOURCE_DIR}/include
)

# contract tests verifying the moku API used by nnue is working as intended
add_executable(moku_contract_tests
    tests/test_moku.cpp
)
target_include_directories(moku_contract_tests PRIVATE ${NNUE_MOKU_INCLUDES})
target_link_libraries(moku_contract_tests PRIVATE
    moku_core
    GTest::gtest_main
    Threads::Threads
)

# run the contract tests at build time; non-zero exit aborts the build
set(MOKU_TESTS_STAMP ${CMAKE_CURRENT_BINARY_DIR}/moku_contract_tests.passed)
add_custom_command(
    OUTPUT ${MOKU_TESTS_STAMP}
    COMMAND moku_contract_tests
    COMMAND ${CMAKE_COMMAND} -E touch ${MOKU_TESTS_STAMP}
    DEPENDS moku_contract_tests
    COMMENT "Verifying moku API contract before building nnue_gen"
    VERBATIM
)
add_custom_target(verify_moku_contract DEPENDS ${MOKU_TESTS_STAMP})

# main executable
add_executable(nnue_gen
    src/main.cpp
    src/nnue.cpp
)
target_include_directories(nnue_gen PRIVATE ${NNUE_MOKU_INCLUDES})
target_link_libraries(nnue_gen PRIVATE
    moku_core
    Threads::Threads
)
target_compile_features(nnue_gen PRIVATE cxx_std_17)

# gate nnue_gen: its compilation only starts once the contract tests pass
add_dependencies(nnue_gen verify_moku_contract)