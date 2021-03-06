# Axel '0vercl0k' Souchet - December 1 2020
cmake_minimum_required(VERSION 3.16.0)
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED True)
set(CMAKE_INTERPROCEDURAL_OPTIMIZATION TRUE)

project(clairvoyance)

include_directories(${CMAKE_CURRENT_LIST_DIR}/third_party/fmt/include)
include_directories(${CMAKE_CURRENT_LIST_DIR}/third_party/kdmp-parser/src/lib)

file(
    GLOB_RECURSE
    clairvoyance_srcfiles
    src/*.cc
)

file(
    GLOB_RECURSE
    kdmp_srcfiles
    ${CMAKE_CURRENT_LIST_DIR}/third_party/kdmp-parser/src/lib/*.cc
)

file(
    GLOB_RECURSE
    fmt_srcfiles
    ${CMAKE_CURRENT_LIST_DIR}/third_party/fmt/src/*.cc
)

add_executable(
    clairvoyance
    ${clairvoyance_srcfiles}
    ${kdmp_srcfiles}
    ${fmt_srcfiles}
)

if (WIN32)
    # https://docs.microsoft.com/en-us/cpp/build/reference/zc-cplusplus
    target_compile_options(
        clairvoyance
        PRIVATE
        "$<$<COMPILE_LANGUAGE:CXX>:/Zc:__cplusplus>"
    )
else (WIN32)
endif(WIN32)
