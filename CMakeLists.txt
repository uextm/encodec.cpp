cmake_minimum_required(VERSION 3.12)
project("encodec" C CXX)

#
# Compile flags
#

if (NOT XCODE AND NOT MSVC AND NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
    set_property(CACHE CMAKE_BUILD_TYPE PROPERTY STRINGS "Debug" "Release" "MinSizeRel" "RelWithDebInfo")
endif()

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)
set(CMAKE_CXX_FLAGS_RELEASE "-O3")

set(THREADS_PREFER_PTHREAD_FLAG ON)

find_package(Threads REQUIRED)

#
# Option list
#

if(CMAKE_SOURCE_DIR STREQUAL CMAKE_CURRENT_SOURCE_DIR)
    set(ENCODEC_STANDALONE ON)
else()
    set(ENCODEC_STANDALONE OFF)
endif()

# general
option(BUILD_SHARED_LIBS        "build shared libraries"        OFF)

# encodec
option(ENCODEC_BUILD_EXAMPLES   "encodec: build examples"       ${ENCODEC_STANDALONE})


if (WIN32)
    add_compile_definitions(_CRT_SECURE_NO_WARNINGS)

    if (BUILD_SHARED_LIBS)
        set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)
    endif()
endif()

if (GGML_CUBLAS)
    add_compile_definitions(GGML_USE_CUBLAS)
endif()

if (GGML_METAL)
    add_compile_definitions(GGML_USE_METAL)
endif()

#
# Build libraries
#

# ggml

set(GGML_LIB ggml)
set(GGML_STATIC_LIB ggml_static)
set(GGML_SHARED_LIB ggml_shared)

add_library(${GGML_LIB} OBJECT
        ggml/src/ggml.c
        ggml/include/ggml/ggml.h
        ggml/src/ggml-alloc.c
        ggml/include/ggml/ggml-alloc.h
        ggml/src/ggml-backend.c
        ggml/include/ggml/ggml-backend.h
        ggml/src/ggml-quants.c
        ggml/src/ggml-quants.h
)

target_include_directories(${GGML_LIB} PUBLIC ./ggml/include/ggml)

target_compile_features   (${GGML_LIB} PUBLIC c_std_11) # don't bump

target_link_libraries(${GGML_LIB} PUBLIC Threads::Threads)

add_library(${GGML_STATIC_LIB} STATIC $<TARGET_OBJECTS:ggml>)

if (BUILD_SHARED_LIBS)
    set_target_properties(${GGML_LIB} PROPERTIES POSITION_INDEPENDENT_CODE ON)

    add_library(${GGML_SHARED_LIB} SHARED $<TARGET_OBJECTS:ggml>)

    target_link_libraries(${GGML_SHARED_LIB} PUBLIC Threads::Threads)

    install(TARGETS ${GGML_SHARED_LIB} LIBRARY)
endif()

# encodec

set(ENCODEC_LIB encodec)
set(ENCODEC_STATIC_LIB encodec_static)
set(ENCODEC_SHARED_LIB encodec_shared)

add_library(${ENCODEC_LIB} OBJECT
        encodec.cpp
        encodec.h
)

target_include_directories(${ENCODEC_LIB} PUBLIC .)
target_include_directories(${ENCODEC_LIB} PUBLIC ./ggml/include/ggml)

target_compile_features   (${ENCODEC_LIB} PUBLIC cxx_std_11)

target_link_libraries(${ENCODEC_LIB} PUBLIC ${GGML_LIB})

add_library(${ENCODEC_STATIC_LIB} STATIC $<TARGET_OBJECTS:encodec>)

if (BUILD_SHARED_LIBS)
    set_target_properties(${ENCODEC_LIB} PROPERTIES POSITION_INDEPENDENT_CODE ON)

    add_library(${ENCODEC_SHARED_LIB} SHARED $<TARGET_OBJECTS:encodec>)

    target_link_libraries(${ENCODEC_SHARED_LIB} PUBLIC ${GGML_LIB})

    install(TARGETS ${ENCODEC_SHARED_LIB} LIBRARY)
endif()

#
# Build examples
#

if (ENCODEC_BUILD_EXAMPLES)
    add_subdirectory(examples)
endif()
