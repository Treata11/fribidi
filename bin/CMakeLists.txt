add_executable(fribidi-bin
    fribidi-main.c
    getopt.c
    getopt1.c
    ${FRIBIDI_UNICODE_VERSION_H}
)
target_include_directories(fribidi-bin PRIVATE ${FRIBIDI_INCLUDE_DIRS})
target_compile_definitions(fribidi-bin PRIVATE HAVE_CONFIG_H ${FRIBIDI_STATIC_CARGS})
target_link_libraries(fribidi-bin PRIVATE fribidi)
add_dependencies(fribidi-bin generate-unicode-version)
if(FRIBIDI_BUILD_BIN)
    install(TARGETS fribidi-bin RUNTIME DESTINATION bin)
endif()

if(NOT FRIBIDI_BUILD_BIN)
    return()
endif()

add_executable(fribidi-benchmark
    fribidi-benchmark.c
    getopt.c
    getopt1.c
    ${FRIBIDI_UNICODE_VERSION_H}
)
target_include_directories(fribidi-benchmark PRIVATE ${FRIBIDI_INCLUDE_DIRS})
target_compile_definitions(fribidi-benchmark PRIVATE HAVE_CONFIG_H ${FRIBIDI_STATIC_CARGS})
target_link_libraries(fribidi-benchmark PRIVATE fribidi)
add_dependencies(fribidi-benchmark generate-unicode-version)

add_executable(fribidi-bidi-types
    fribidi-bidi-types.c
    ${FRIBIDI_UNICODE_VERSION_H}
)
target_include_directories(fribidi-bidi-types PRIVATE ${FRIBIDI_INCLUDE_DIRS})
target_compile_definitions(fribidi-bidi-types PRIVATE HAVE_CONFIG_H ${FRIBIDI_STATIC_CARGS})
target_link_libraries(fribidi-bidi-types PRIVATE fribidi)
add_dependencies(fribidi-bidi-types generate-unicode-version)

add_executable(fribidi-caprtl2utf8
    fribidi-caprtl2utf8.c
    ${FRIBIDI_UNICODE_VERSION_H}
)
target_include_directories(fribidi-caprtl2utf8 PRIVATE ${FRIBIDI_INCLUDE_DIRS})
target_compile_definitions(fribidi-caprtl2utf8 PRIVATE HAVE_CONFIG_H ${FRIBIDI_STATIC_CARGS})
target_link_libraries(fribidi-caprtl2utf8 PRIVATE fribidi)
add_dependencies(fribidi-caprtl2utf8 generate-unicode-version)

if(FRIBIDI_FUZZER_LDFLAGS)
    add_executable(fribidi-fuzzer
        fribidi-fuzzer.c
        ${FRIBIDI_UNICODE_VERSION_H}
    )
    target_include_directories(fribidi-fuzzer PRIVATE ${FRIBIDI_INCLUDE_DIRS})
    target_compile_definitions(fribidi-fuzzer PRIVATE HAVE_CONFIG_H ${FRIBIDI_STATIC_CARGS})
    target_link_libraries(fribidi-fuzzer PRIVATE fribidi)
    target_link_options(fribidi-fuzzer PRIVATE ${FRIBIDI_FUZZER_LDFLAGS})
    add_dependencies(fribidi-fuzzer generate-unicode-version)
endif()