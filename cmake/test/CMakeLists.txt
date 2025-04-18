# This cmake build script is meant for verifying the various CMake configuration script.

cmake_minimum_required(VERSION 3.12...3.28)
project(sdl_test LANGUAGES C)

include(GenerateExportHeader)

cmake_policy(SET CMP0074 NEW)

# Override CMAKE_FIND_ROOT_PATH_MODE to allow search for SDL2 outside of sysroot
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE NEVER)

include(FeatureSummary)

option(TEST_SHARED "Test linking to shared SDL2 library" ON)
add_feature_info("TEST_SHARED" TEST_SHARED "Test linking with shared library")

option(TEST_STATIC "Test linking to static SDL2 library" ON)
add_feature_info("TEST_STATIC" TEST_STATIC "Test linking with static library")

if(TEST_SHARED)
    find_package(SDL2 REQUIRED CONFIG COMPONENTS SDL2)
    if(EMSCRIPTEN OR WIN32)
        find_package(SDL2 REQUIRED CONFIG COMPONENTS SDL2main)
    endif()
    add_executable(gui-shared WIN32 main_gui.c)
    if(TARGET SDL2::SDL2main)
        target_link_libraries(gui-shared PRIVATE SDL2::SDL2main)
    endif()
    target_link_libraries(gui-shared PRIVATE SDL2::SDL2)
    if(WIN32)
        add_custom_command(TARGET gui-shared POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E copy_if_different "$<TARGET_FILE:SDL2::SDL2>" "$<TARGET_FILE_DIR:gui-shared>"
        )
    endif()

    add_library(sharedlib-shared SHARED main_lib.c)
    target_link_libraries(sharedlib-shared PRIVATE SDL2::SDL2)
    generate_export_header(sharedlib-shared EXPORT_MACRO_NAME MYLIBRARY_EXPORT)
    target_compile_definitions(sharedlib-shared PRIVATE "EXPORT_HEADER=\"${CMAKE_CURRENT_BINARY_DIR}/sharedlib-shared_export.h\"")
    set_target_properties(sharedlib-shared PROPERTIES C_VISIBILITY_PRESET "hidden")

    add_executable(gui-shared-vars WIN32 main_gui.c)
    target_link_libraries(gui-shared-vars PRIVATE ${SDL2_LIBRARIES})
    target_include_directories(gui-shared-vars PRIVATE ${SDL2_INCLUDE_DIRS})

    add_executable(cli-shared main_cli.c)
    target_link_libraries(cli-shared PRIVATE SDL2::SDL2)
    if(WIN32)
        add_custom_command(TARGET cli-shared POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E copy_if_different "$<TARGET_FILE:SDL2::SDL2>" "$<TARGET_FILE_DIR:cli-shared>"
        )
    endif()

    # SDL2_LIBRARIES does not support creating a cli SDL2 application
    # (it is possible that SDL2main is a stub, but we don't know for sure)
    if(NOT TARGET SDL2::SDL2main)
        add_executable(cli-shared-vars main_cli.c)
        target_link_libraries(cli-shared-vars PRIVATE ${SDL2_LIBRARIES})
        target_include_directories(cli-shared-vars PRIVATE ${SDL2_INCLUDE_DIRS})
    endif()

    add_library(sharedlib-shared-vars SHARED main_lib.c)
    target_link_libraries(sharedlib-shared-vars PRIVATE ${SDL2_LIBRARIES})
    target_include_directories(sharedlib-shared-vars PRIVATE ${SDL2_INCLUDE_DIRS})
    generate_export_header(sharedlib-shared-vars EXPORT_MACRO_NAME MYLIBRARY_EXPORT)
    target_compile_definitions(sharedlib-shared-vars PRIVATE "EXPORT_HEADER=\"${CMAKE_CURRENT_BINARY_DIR}/sharedlib-shared-vars_export.h\"")
    set_target_properties(sharedlib-shared-vars PROPERTIES C_VISIBILITY_PRESET "hidden")
endif()

if(TEST_STATIC)
    find_package(SDL2 REQUIRED CONFIG COMPONENTS SDL2-static)
    if(EMSCRIPTEN OR WIN32)
        find_package(SDL2 REQUIRED CONFIG COMPONENTS SDL2main)
    endif()
    add_executable(gui-static WIN32 main_gui.c)
    if(TARGET SDL2::SDL2main)
        target_link_libraries(gui-static PRIVATE SDL2::SDL2main)
    endif()
    target_link_libraries(gui-static PRIVATE SDL2::SDL2-static)

    option(SDL_STATIC_PIC "SDL static library has been built with PIC")
    if(SDL_STATIC_PIC OR WIN32)
        add_library(sharedlib-static SHARED main_lib.c)
        target_link_libraries(sharedlib-static PRIVATE SDL2::SDL2-static)
        generate_export_header(sharedlib-static EXPORT_MACRO_NAME MYLIBRARY_EXPORT)
        target_compile_definitions(sharedlib-static PRIVATE "EXPORT_HEADER=\"${CMAKE_CURRENT_BINARY_DIR}/sharedlib-static_export.h\"")
        set_target_properties(sharedlib-static PROPERTIES C_VISIBILITY_PRESET "hidden")
    endif()

    add_executable(gui-static-vars WIN32 main_gui.c)
    target_link_libraries(gui-static-vars PRIVATE ${SDL2MAIN_LIBRARY} ${SDL2_STATIC_LIBRARIES})
    target_include_directories(gui-static-vars PRIVATE ${SDL2_INCLUDE_DIRS})

    add_executable(cli-static main_cli.c)
    target_link_libraries(cli-static PRIVATE SDL2::SDL2-static)

    # SDL2_LIBRARIES does not support creating a cli SDL2 application (when SDL2::SDL2main is available)
    # (it is possible that SDL2main is a stub, but we don't know for sure)
    if(NOT TARGET SDL2::SDL2main)
        add_executable(cli-static-vars main_cli.c)
        target_link_libraries(cli-static-vars PRIVATE ${SDL2_STATIC_LIBRARIES})
        target_include_directories(cli-static-vars PRIVATE ${SDL2_INCLUDE_DIRS})
    endif()
endif()

message(STATUS "SDL2_PREFIX:            ${SDL2_PREFIX}")
message(STATUS "SDL2_INCLUDE_DIR:       ${SDL2_INCLUDE_DIR}")
message(STATUS "SDL2_INCLUDE_DIRS:      ${SDL2_INCLUDE_DIRS}")
message(STATUS "SDL2_LIBRARIES:         ${SDL2_LIBRARIES}")
message(STATUS "SDL2_STATIC_LIBRARIES:  ${SDL2_STATIC_LIBRARIES}")
message(STATUS "SDL2MAIN_LIBRARY:       ${SDL2MAIN_LIBRARY}")
message(STATUS "SDL2TEST_LIBRARY:       ${SDL2TEST_LIBRARY}")

feature_summary(WHAT ALL)
