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

# Generate fribidi-unicode-version.h at configuration time
find_program(CC_COMPILER ${CMAKE_C_COMPILER})
if(NOT CC_COMPILER)
    message(FATAL_ERROR "C compiler not found")
endif()

# Include directories matching FRIBIDI_INCLUDE_DIRS
set(GEN_INCLUDE_DIRS 
    -I${CMAKE_SOURCE_DIR}           # .../fribidi/
    -I${CMAKE_SOURCE_DIR}/lib       # .../fribidi/lib/ (for common.h)
    -I${CMAKE_SOURCE_DIR}/gen.tab   # .../fribidi/gen.tab/
    -I${CMAKE_BINARY_DIR}           # .../build_macOS/ (for config.h)
    -I${CMAKE_BINARY_DIR}/lib       # .../build_macOS/lib/ (for fribidi-config.h)
    -I${CMAKE_BINARY_DIR}/gen.tab   # .../build_macOS/gen.tab/ (for generated files)
)

# Add macOS SDK sysroot
if(APPLE)
    # Use Xcode's default macOS SDK
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

# Empty target for compatibility
add_custom_target(generate-unicode-version)

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

# Generate tab files during build
foreach(TAB_NAME ${TAB_NAMES})
    set(GEN_PROG_NAME "gen-${TAB_NAME}-tab")
    set(GEN_OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/${TAB_NAME}.tab.i")

    file(TOUCH ${GEN_OUTPUT})
    set_source_files_properties(${GEN_OUTPUT} PROPERTIES GENERATED TRUE)

    add_executable(${GEN_PROG_NAME} gen-${TAB_NAME}-tab.c packtab.c)
    target_include_directories(${GEN_PROG_NAME} PRIVATE ${FRIBIDI_INCLUDE_DIRS})
    target_compile_definitions(${GEN_PROG_NAME} PRIVATE 
        ${NATIVE_C_ARGS}
        HAVE_CONFIG_H=0
    )

    add_custom_command(
        OUTPUT ${GEN_OUTPUT}
        COMMAND ${CMAKE_CURRENT_BINARY_DIR}/${GEN_PROG_NAME}
            ${COMPRESSION} 
            ${TAB_FILES_${TAB_NAME}}
            > ${GEN_OUTPUT}
        DEPENDS ${GEN_PROG_NAME} ${TAB_FILES_${TAB_NAME}}
        COMMENT "Generating ${TAB_NAME}.tab.i"
        VERBATIM
    )
    list(APPEND GENERATED_TAB_INCLUDE_FILES ${GEN_OUTPUT})
endforeach()

add_custom_target(generate-tabs DEPENDS ${GENERATED_TAB_INCLUDE_FILES})

# Export variables to parent scope
set(FRIBIDI_UNICODE_VERSION_H ${FRIBIDI_UNICODE_VERSION_H} PARENT_SCOPE)
set(GENERATED_TAB_INCLUDE_FILES ${GENERATED_TAB_INCLUDE_FILES} PARENT_SCOPE)