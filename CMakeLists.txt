cmake_minimum_required(VERSION 3.20)
project(minijs,LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

option(MINIJS_SANITIZE "Build with AddressSanitizer and UBSan" OFF)
if(MINIJS_SANITIZE)
    add_compile_options(-fsanitize=address,undefined -fno-omit-frame-pointer)
    add_link_options(-fsanitize=address,undefined)
endif()

add_library(minijs_core
    src/engine.cpp
)
target_include_directories(minijs_core PUBLIC include)
target_compile_options(minijs_core PRIVATE -Wall -Wextra -Wpedantic)

if(NOT EMSCRIPTEN)
    add_executable(minijs src/main.cpp)
    target_link_libraries(minijs PRIVATE minijs_core)

    enable_testing()
    add_executable(minijs_tests
        tests/test_main.cpp
        tests/engine_tests.cpp
    )
    target_link_libraries(minijs_tests PRIVATE minijs_core)
    add_test(NAME minijs_tests COMMAND minijs_tests
             WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})
endif()
