# Native build settings
set(NATIVE_C_ARGS 
    -DHAVE_STRINGIZE 
    -DDONT_HAVE_FRIBIDI_CONFIG_H
)
check_include_file(stdlib.h HAVE_STDLIB_H_NATIVE)
if(HAVE_STDLIB_H_NATIVE)
    list(APPEND NATIVE_C_ARGS -DHAVE_STDLIB_H)
endif()
check_include_file(string.h HAVE_STRING_H_NATIVE)
if(HAVE_STRING_H_NATIVE)
    list(APPEND NATIVE_C_ARGS -DHAVE_STRING_H)
endif()
check_include_file(strings.h HAVE_STRINGS_H_NATIVE)
if(HAVE_STRINGS_H_NATIVE)
    list(APPEND NATIVE_C_ARGS -DHAVE_STRINGS_H -DSTDC_HEADERS=1)
endif()

# Define output paths
set(FRIBIDI_UNICODE_VERSION_H ${CMAKE_BINARY_DIR}/fribidi-unicode-version.h)
set(GENERATED_TAB_INCLUDE_FILES "")

# Generate fribidi-unicode-version.h and tab files at configuration time for cross-compilation
find_program(CC_COMPILER ${CMAKE_C_COMPILER})
if(NOT CC_COMPILER)
    message(FATAL_ERROR "C compiler not found")
endif()

# Include directories matching FRIBIDI_INCLUDE_DIRS
set(GEN_INCLUDE_DIRS 
    -I${CMAKE_SOURCE_DIR}           # .../fribidi/
    -I${CMAKE_SOURCE_DIR}/lib       # .../fribidi/lib/ (for common.h)
    -I${CMAKE_SOURCE_DIR}/gen.tab   # .../fribidi/gen.tab/
    -I${CMAKE_BINARY_DIR}           # .../build_iOS/ (for config.h)
    -I${CMAKE_BINARY_DIR}/lib       # .../build_iOS/lib/ (for fribidi-config.h)
    -I${CMAKE_BINARY_DIR}/gen.tab   # .../build_iOS/gen.tab/ (for generated files)
)

# Add macOS SDK sysroot for host compilation
if(APPLE)
    execute_process(
        COMMAND xcrun --sdk macosx --show-sdk-path
        OUTPUT_VARIABLE MACOS_SDK_PATH
        OUTPUT_STRIP_TRAILING_WHITESPACE
    )
    if(NOT MACOS_SDK_PATH)
        message(FATAL_ERROR "Failed to find macOS SDK path")
    endif()
    list(APPEND GEN_C_FLAGS -isysroot ${MACOS_SDK_PATH})
endif()

# Debug: Print compiler flags
message("GEN_C_FLAGS: ${GEN_C_FLAGS}")
message("GEN_INCLUDE_DIRS: ${GEN_INCLUDE_DIRS}")

# Compile gen-unicode-version during configuration
set(GEN_UNICODE_SRC ${CMAKE_SOURCE_DIR}/gen.tab/gen-unicode-version.c)
set(GEN_UNICODE_BIN ${CMAKE_BINARY_DIR}/gen-unicode-version-temp)
execute_process(
    COMMAND ${CC_COMPILER} ${GEN_C_FLAGS} ${NATIVE_C_ARGS} -DHAVE_CONFIG_H=0
            ${GEN_INCLUDE_DIRS}
            ${GEN_UNICODE_SRC} -o ${GEN_UNICODE_BIN}
    RESULT_VARIABLE COMPILE_RESULT
    OUTPUT_VARIABLE COMPILE_OUTPUT
    ERROR_VARIABLE COMPILE_ERROR
)
if(NOT COMPILE_RESULT EQUAL 0)
    message(STATUS "Compile command: ${CC_COMPILER} ${GEN_C_FLAGS} ${NATIVE_C_ARGS} -DHAVE_CONFIG_H=0 ${GEN_INCLUDE_DIRS} ${GEN_UNICODE_SRC} -o ${GEN_UNICODE_BIN}")
    message(STATUS "Compile output: ${COMPILE_OUTPUT}")
    message(STATUS "Compile error: ${COMPILE_ERROR}")
    message(FATAL_ERROR "Failed to compile gen-unicode-version")
endif()

# Run it to generate the header
execute_process(
    COMMAND ${GEN_UNICODE_BIN}
            ${CMAKE_SOURCE_DIR}/gen.tab/unidata/ReadMe.txt 
            ${CMAKE_SOURCE_DIR}/gen.tab/unidata/BidiMirroring.txt
    OUTPUT_FILE ${FRIBIDI_UNICODE_VERSION_H}
    RESULT_VARIABLE RUN_RESULT
    OUTPUT_VARIABLE RUN_OUTPUT
    ERROR_VARIABLE RUN_ERROR
)
if(NOT RUN_RESULT EQUAL 0)
    message(STATUS "Run output: ${RUN_OUTPUT}")
    message(STATUS "Run error: ${RUN_ERROR}")
    message(FATAL_ERROR "Failed to run gen-unicode-version")
endif()

# Clean up temporary binary
file(REMOVE ${GEN_UNICODE_BIN})

# Mark as generated
set_source_files_properties(${FRIBIDI_UNICODE_VERSION_H} PROPERTIES GENERATED TRUE)
install(FILES ${FRIBIDI_UNICODE_VERSION_H} DESTINATION include/fribidi)

set(COMPRESSION 2)

# Define tabs explicitly
set(TAB_NAMES 
    "bidi-type" 
    "joining-type" 
    "arabic-shaping" 
    "mirroring" 
    "brackets" 
    "brackets-type"
)

set(TAB_FILES_bidi-type "${CMAKE_SOURCE_DIR}/gen.tab/unidata/UnicodeData.txt")
set(TAB_FILES_joining-type "${CMAKE_SOURCE_DIR}/gen.tab/unidata/UnicodeData.txt;${CMAKE_SOURCE_DIR}/gen.tab/unidata/ArabicShaping.txt")
set(TAB_FILES_arabic-shaping "${CMAKE_SOURCE_DIR}/gen.tab/unidata/UnicodeData.txt")
set(TAB_FILES_mirroring "${CMAKE_SOURCE_DIR}/gen.tab/unidata/BidiMirroring.txt")
set(TAB_FILES_brackets "${CMAKE_SOURCE_DIR}/gen.tab/unidata/BidiBrackets.txt;${CMAKE_SOURCE_DIR}/gen.tab/unidata/UnicodeData.txt")
set(TAB_FILES_brackets-type "${CMAKE_SOURCE_DIR}/gen.tab/unidata/BidiBrackets.txt")

# Precompile and run tab generators during configuration
foreach(TAB_NAME ${TAB_NAMES})
    set(GEN_PROG_NAME "gen-${TAB_NAME}-tab")
    set(GEN_OUTPUT "${CMAKE_BINARY_DIR}/gen.tab/${TAB_NAME}.tab.i")
    set(GEN_PROG_SRC "${CMAKE_SOURCE_DIR}/gen.tab/gen-${TAB_NAME}-tab.c;${CMAKE_SOURCE_DIR}/gen.tab/packtab.c")
    set(GEN_PROG_BIN "${CMAKE_BINARY_DIR}/${GEN_PROG_NAME}-temp")

    # Compile the generator
    execute_process(
        COMMAND ${CC_COMPILER} ${GEN_C_FLAGS} ${NATIVE_C_ARGS} -DHAVE_CONFIG_H=0
                ${GEN_INCLUDE_DIRS}
                ${GEN_PROG_SRC} -o ${GEN_PROG_BIN}
        RESULT_VARIABLE COMPILE_RESULT
        OUTPUT_VARIABLE COMPILE_OUTPUT
        ERROR_VARIABLE COMPILE_ERROR
    )
    if(NOT COMPILE_RESULT EQUAL 0)
        message(STATUS "Compile command: ${CC_COMPILER} ${GEN_C_FLAGS} ${NATIVE_C_ARGS} -DHAVE_CONFIG_H=0 ${GEN_INCLUDE_DIRS} ${GEN_PROG_SRC} -o ${GEN_PROG_BIN}")
        message(STATUS "Compile output: ${COMPILE_OUTPUT}")
        message(STATUS "Compile error: ${COMPILE_ERROR}")
        message(FATAL_ERROR "Failed to compile ${GEN_PROG_NAME}")
    endif()

    # Run it to generate the tab file
    execute_process(
        COMMAND ${GEN_PROG_BIN}
                ${COMPRESSION} 
                ${TAB_FILES_${TAB_NAME}}
        OUTPUT_FILE ${GEN_OUTPUT}
        RESULT_VARIABLE RUN_RESULT
        OUTPUT_VARIABLE RUN_OUTPUT
        ERROR_VARIABLE RUN_ERROR
    )
    if(NOT RUN_RESULT EQUAL 0)
        message(STATUS "Run output: ${RUN_OUTPUT}")
        message(STATUS "Run error: ${RUN_ERROR}")
        message(FATAL_ERROR "Failed to run ${GEN_PROG_NAME}")
    endif()

    # Clean up temporary binary
    file(REMOVE ${GEN_PROG_BIN})

    set_source_files_properties(${GEN_OUTPUT} PROPERTIES GENERATED TRUE)
    list(APPEND GENERATED_TAB_INCLUDE_FILES ${GEN_OUTPUT})
endforeach()

# Empty targets for compatibility
add_custom_target(generate-unicode-version)
add_custom_target(generate-tabs)

# Export variables to parent scope
set(FRIBIDI_UNICODE_VERSION_H ${FRIBIDI_UNICODE_VERSION_H} PARENT_SCOPE)
set(GENERATED_TAB_INCLUDE_FILES ${GENERATED_TAB_INCLUDE_FILES} PARENT_SCOPE)