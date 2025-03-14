cmake_minimum_required(VERSION 3.18)
project(fribidi 
    VERSION 1.0.16 
    LANGUAGES C
)

# Versioning
set(INTERFACE_AGE 0)
set(INTERFACE_VERSION 4)
set(SOVERSION 0)
set(LIBVERSION "${SOVERSION}.${INTERFACE_VERSION}.0")

# Compiler setup
include(CheckFunctionExists)
include(CheckIncludeFile)

# Configuration options
option(FRIBIDI_BUILD_DEPRECATED "Build deprecated functionality" ON)
option(FRIBIDI_BUILD_DOCS "Build documentation" ON)
option(FRIBIDI_BUILD_BIN "Build binaries" ON)
option(FRIBIDI_BUILD_TESTS "Build tests" ON)
set(FRIBIDI_FUZZER_LDFLAGS "" CACHE STRING "Extra LDFLAGS for fuzzing binaries")

# Handle Windows symbol visibility
set(FRIBIDI_STATIC_CARGS "")
set(FRIBIDI_BUILD_CARGS "")
if(WIN32)
    set(FRIBIDI_BUILD_CARGS -DFRIBIDI_BUILD)
    if(CMAKE_BUILD_TYPE STREQUAL "Static")
        set(FRIBIDI_STATIC_CARGS -DFRIBIDI_LIB_STATIC)
    endif()
endif()

# Generate config.h
set(HAVE_MEMMOVE 0)
set(HAVE_MEMSET 0)
set(HAVE_STRDUP 0)
check_function_exists(memmove HAVE_MEMMOVE)
check_function_exists(memset HAVE_MEMSET)
check_function_exists(strdup HAVE_STRDUP)

set(HAVE_STDLIB_H 0)
set(HAVE_STRING_H 0)
set(HAVE_MEMORY_H 0)
set(HAVE_STRINGS_H 0)
set(HAVE_SYS_TIMES_H 0)
check_include_file(stdlib.h HAVE_STDLIB_H)
check_include_file(string.h HAVE_STRING_H)
check_include_file(memory.h HAVE_MEMORY_H)
check_include_file(strings.h HAVE_STRINGS_H)
check_include_file(sys/times.h HAVE_SYS_TIMES_H)

if(HAVE_STRINGS_H)
    set(STDC_HEADERS 1)
endif()
set(HAVE_STRINGIZE 1)

if(CMAKE_BUILD_TYPE MATCHES Debug)
    set(DEBUG 1)
endif()
if(NOT FRIBIDI_BUILD_DEPRECATED)
    set(FRIBIDI_NO_DEPRECATED 1)
endif()

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/config.h.in ${CMAKE_CURRENT_BINARY_DIR}/config.h)

# Include directories
set(FRIBIDI_INCLUDE_DIRS 
    ${CMAKE_CURRENT_SOURCE_DIR} 
    ${CMAKE_CURRENT_SOURCE_DIR}/lib 
    ${CMAKE_CURRENT_SOURCE_DIR}/gen.tab
    ${CMAKE_CURRENT_BINARY_DIR}
)

# Subdirectories
add_subdirectory(gen.tab)
add_subdirectory(lib)
if(FRIBIDI_BUILD_BIN OR FRIBIDI_BUILD_TESTS)
    add_subdirectory(bin)
endif()

# Pkg-config file
find_package(PkgConfig)
if(PKG_CONFIG_FOUND)
    set(prefix "${CMAKE_INSTALL_PREFIX}")
    set(exec_prefix "${CMAKE_INSTALL_PREFIX}")
    set(libdir "${CMAKE_INSTALL_PREFIX}/lib")
    set(includedir "${CMAKE_INSTALL_PREFIX}/include")
    set(PACKAGE_NAME "GNU FriBidi")
    set(PACKAGE "fribidi")
    set(VERSION "${PROJECT_VERSION}")
    set(FRIBIDI_CPPFLAGS "${FRIBIDI_STATIC_CARGS}")
    set(ENABLE_SHARED_TRUE "") # Assume shared by default
    set(ENABLE_SHARED_FALSE "#") # Comment out static flags
    configure_file(${CMAKE_CURRENT_SOURCE_DIR}/fribidi.pc.in ${CMAKE_CURRENT_BINARY_DIR}/fribidi.pc @ONLY)
    install(FILES ${CMAKE_CURRENT_BINARY_DIR}/fribidi.pc DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)
endif()