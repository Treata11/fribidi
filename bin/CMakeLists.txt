# Helper function to avoid repetition
function(add_fribidi_executable TARGET_NAME SOURCES)
    add_executable(${TARGET_NAME} ${SOURCES} ${FRIBIDI_UNICODE_VERSION_H} ${CMAKE_BINARY_DIR}/lib/fribidi-config.h)
    target_include_directories(${TARGET_NAME} PRIVATE 
        ${CMAKE_BINARY_DIR}/lib
        ${FRIBIDI_INCLUDE_DIRS}
    )
    target_compile_definitions(${TARGET_NAME} PRIVATE 
        HAVE_CONFIG_H 
        ${FRIBIDI_STATIC_CARGS}
        FRIBIDI="FRIBIDI_VERSION"
        FRIBIDI_BUGREPORT="https://github.com/fribidi/fribidi/issues/new"
    )
    target_compile_options(${TARGET_NAME} PRIVATE 
        -include ${CMAKE_BINARY_DIR}/lib/fribidi-config.h
    )
    target_link_libraries(${TARGET_NAME} PRIVATE fribidi)
    add_dependencies(${TARGET_NAME} generate-unicode-version)
    if(FRIBIDI_BUILD_BIN)
        install(TARGETS ${TARGET_NAME} RUNTIME DESTINATION bin)
    endif()
endfunction()

# Main binary
add_fribidi_executable(fribidi-bin "fribidi-main.c;getopt.c;getopt1.c")

if(NOT FRIBIDI_BUILD_BIN)
    return()
endif()

# Additional binaries
add_fribidi_executable(fribidi-benchmark "fribidi-benchmark.c;getopt.c;getopt1.c")
add_fribidi_executable(fribidi-bidi-types "fribidi-bidi-types.c")
add_fribidi_executable(fribidi-caprtl2utf8 "fribidi-caprtl2utf8.c")

if(FRIBIDI_FUZZER_LDFLAGS)
    add_fribidi_executable(fribidi-fuzzer "fribidi-fuzzer.c")
    target_link_options(fribidi-fuzzer PRIVATE ${FRIBIDI_FUZZER_LDFLAGS})
endif()