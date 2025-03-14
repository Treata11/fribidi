# Native build settings
set(NATIVE_C_ARGS -UHAVE_CONFIG_H -DHAVE_STRINGIZE -DDONT_HAVE_FRIBIDI_CONFIG_H)
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

# Create placeholder files during configuration to satisfy CMake
file(TOUCH ${FRIBIDI_UNICODE_VERSION_H})
set_source_files_properties(${FRIBIDI_UNICODE_VERSION_H} PROPERTIES GENERATED TRUE)
install(FILES ${FRIBIDI_UNICODE_VERSION_H} DESTINATION include/fribidi)

# Build gen-unicode-version
add_executable(gen-unicode-version gen-unicode-version.c)
target_include_directories(gen-unicode-version PRIVATE ${FRIBIDI_INCLUDE_DIRS})
target_compile_definitions(gen-unicode-version PRIVATE ${NATIVE_C_ARGS})

# Generate fribidi-unicode-version.h during build
add_custom_command(
    OUTPUT ${FRIBIDI_UNICODE_VERSION_H}.build
    COMMAND ${CMAKE_CURRENT_BINARY_DIR}/gen-unicode-version
        ${CMAKE_CURRENT_SOURCE_DIR}/unidata/ReadMe.txt 
        ${CMAKE_CURRENT_SOURCE_DIR}/unidata/BidiMirroring.txt 
        gen-unicode-version
        > ${FRIBIDI_UNICODE_VERSION_H}
    DEPENDS gen-unicode-version 
        ${CMAKE_CURRENT_SOURCE_DIR}/unidata/ReadMe.txt 
        ${CMAKE_CURRENT_SOURCE_DIR}/unidata/BidiMirroring.txt
    COMMENT "Generating fribidi-unicode-version.h"
)
add_custom_target(generate-unicode-version 
    DEPENDS ${FRIBIDI_UNICODE_VERSION_H}.build
)

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

set(TAB_FILES_bidi-type "${CMAKE_CURRENT_SOURCE_DIR}/unidata/UnicodeData.txt")
set(TAB_FILES_joining-type "${CMAKE_CURRENT_SOURCE_DIR}/unidata/UnicodeData.txt;${CMAKE_CURRENT_SOURCE_DIR}/unidata/ArabicShaping.txt")
set(TAB_FILES_arabic-shaping "${CMAKE_CURRENT_SOURCE_DIR}/unidata/UnicodeData.txt")
set(TAB_FILES_mirroring "${CMAKE_CURRENT_SOURCE_DIR}/unidata/BidiMirroring.txt")
set(TAB_FILES_brackets "${CMAKE_CURRENT_SOURCE_DIR}/unidata/BidiBrackets.txt;${CMAKE_CURRENT_SOURCE_DIR}/unidata/UnicodeData.txt")
set(TAB_FILES_brackets-type "${CMAKE_CURRENT_SOURCE_DIR}/unidata/BidiBrackets.txt")

# Generate tab files
foreach(TAB_NAME ${TAB_NAMES})
    set(GEN_PROG_NAME "gen-${TAB_NAME}-tab")
    set(GEN_OUTPUT "${CMAKE_CURRENT_BINARY_DIR}/${TAB_NAME}.tab.i")

    # Create placeholder file
    file(TOUCH ${GEN_OUTPUT})
    set_source_files_properties(${GEN_OUTPUT} PROPERTIES GENERATED TRUE)

    add_executable(${GEN_PROG_NAME} gen-${TAB_NAME}-tab.c packtab.c)
    target_include_directories(${GEN_PROG_NAME} PRIVATE ${FRIBIDI_INCLUDE_DIRS})
    target_compile_definitions(${GEN_PROG_NAME} PRIVATE ${NATIVE_C_ARGS})

    add_custom_command(
        OUTPUT ${GEN_OUTPUT}.build
        COMMAND ${CMAKE_CURRENT_BINARY_DIR}/${GEN_PROG_NAME}
            ${COMPRESSION} 
            ${TAB_FILES_${TAB_NAME}} 
            ${GEN_PROG_NAME}
            > ${GEN_OUTPUT}
        DEPENDS ${GEN_PROG_NAME} ${TAB_FILES_${TAB_NAME}}
        COMMENT "Generating ${TAB_NAME}.tab.i"
    )
    list(APPEND GENERATED_TAB_INCLUDE_FILES ${GEN_OUTPUT})
endforeach()

add_custom_target(generate-tabs DEPENDS 
    ${GENERATED_TAB_INCLUDE_FILES_BUILD}
)

# Export variables to parent scope
set(FRIBIDI_UNICODE_VERSION_H ${FRIBIDI_UNICODE_VERSION_H} PARENT_SCOPE)
set(GENERATED_TAB_INCLUDE_FILES ${GENERATED_TAB_INCLUDE_FILES} PARENT_SCOPE)