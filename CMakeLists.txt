# NOTE:  If you installed SDL3 at an unusual place and SDL3Config fails,
#        you can define SDL3_INCLUDE_DIRS on the cmdline.  For example:
#        cmake -DSDL3_INCLUDE_DIRS=/opt/SDL3/include [other stuff ....]

cmake_minimum_required(VERSION 3.0...3.28)
project(sdl2_compat VERSION 2.32.57 LANGUAGES C)

if(POLICY CMP0074)
  # CMP0074: find_package() uses <PackageName>_ROOT variables.
  cmake_policy(SET CMP0074 NEW)
endif()

if(CMAKE_SOURCE_DIR STREQUAL PROJECT_SOURCE_DIR)
  set(SDL2COMPAT_MAINPROJECT ON)
  set(SDL2COMPAT_SUBPROJECT OFF)
else()
  set(SDL2COMPAT_MAINPROJECT OFF)
  set(SDL2COMPAT_SUBPROJECT ON)
endif()

if(CMAKE_VERSION VERSION_LESS 3.16.0 OR SDL2COMPAT_SUBPROJECT)
  # - CMake versions <3.16 do not support the OBJC language
  # - When SDL is built as a subproject and when the main project does not enable OBJC,
  #   CMake fails due to missing internal CMake variables (CMAKE_OBJC_COMPILE_OBJECT)
  #   (reproduced with CMake 3.24.2)
else()
  if(APPLE)
    enable_language(OBJC)
  endif()
endif()

set(SDL_MAJOR_VERSION ${sdl2_compat_VERSION_MAJOR})
set(SDL_MINOR_VERSION ${sdl2_compat_VERSION_MINOR})
set(SDL_MICRO_VERSION ${sdl2_compat_VERSION_PATCH})
set(SDL_VERSION "${SDL_MAJOR_VERSION}.${SDL_MINOR_VERSION}.${SDL_MICRO_VERSION}")

include(CheckCSourceCompiles)
include(CheckIncludeFile)
include(CheckCCompilerFlag)
include(CheckLanguage)
include(CMakeDependentOption)
include(CMakePackageConfigHelpers)
include(CMakeParseArguments)
include(CMakePushCheckState)
include(GNUInstallDirs)
include("${CMAKE_CURRENT_LIST_DIR}/cmake/CheckCPUArchitecture.cmake")
include("${CMAKE_CURRENT_LIST_DIR}/cmake/macros.cmake")
include("${CMAKE_CURRENT_LIST_DIR}/cmake/sdlchecks.cmake")
include("${CMAKE_CURRENT_LIST_DIR}/cmake/sdlcompilers.cmake")
include("${CMAKE_CURRENT_LIST_DIR}/cmake/sdlplatform.cmake")

# Calculate libtool-compatible SO version
set(SDL_SO_VERSION_MAJOR "0")
if(SDL_MINOR_VERSION MATCHES "[02468]$")
  # Stable branch, 2.24.1 -> libSDL2-2.0.so.0.2400.1
  math(EXPR SDL_SO_VERSION_MINOR "${SDL_MINOR_VERSION} * 100")
  set(SDL_SO_VERSION_MICRO ${SDL_MICRO_VERSION})
else()
  # Development branch, 2.23.1 -> libSDL2-2.0.so.0.2301.0
  math(EXPR SDL_SO_VERSION_MINOR "${SDL_MINOR_VERSION} * 100 + ${SDL_MICRO_VERSION}")
  set(SDL_SO_VERSION_MICRO 0)
endif()
set(SDL_SO_VERSION "${SDL_SO_VERSION_MAJOR}.${SDL_SO_VERSION_MINOR}.${SDL_SO_VERSION_MICRO}")

if(SDL_MINOR_VERSION MATCHES "[02468]$")
  math(EXPR SDL_DYLIB_COMPAT_VERSION_MAJOR "100 * ${SDL_MINOR_VERSION} + 1")
  set(SDL_DYLIB_COMPAT_VERSION_MINOR "0")
  math(EXPR SDL_DYLIB_CURRENT_VERSION_MAJOR "${SDL_DYLIB_COMPAT_VERSION_MAJOR}")
  set(SDL_DYLIB_CURRENT_VERSION_MINOR "${SDL_MICRO_VERSION}")
else()
  math(EXPR SDL_DYLIB_COMPAT_VERSION_MAJOR "100 * ${SDL_MINOR_VERSION} + ${SDL_MICRO_VERSION} + 1")
  set(SDL_DYLIB_COMPAT_VERSION_MINOR "0")
  math(EXPR SDL_DYLIB_CURRENT_VERSION_MAJOR "${SDL_DYLIB_COMPAT_VERSION_MAJOR}")
  set(SDL_DYLIB_CURRENT_VERSION_MINOR "0")
endif()
set(SDL_DYLIB_COMPAT_VERSION_MICRO "0")
set(SDL_DYLIB_CURRENT_VERSION_MICRO "0")

set(SDL_DYLIB_CURRENT_VERSION "${SDL_DYLIB_CURRENT_VERSION_MAJOR}.${SDL_DYLIB_CURRENT_VERSION_MINOR}.${SDL_DYLIB_CURRENT_VERSION_MICRO}")
set(SDL_DYLIB_COMPAT_VERSION "${SDL_DYLIB_COMPAT_VERSION_MAJOR}.${SDL_DYLIB_COMPAT_VERSION_MINOR}.${SDL_DYLIB_COMPAT_VERSION_MICRO}")

option(SDL2COMPAT_TESTS "Enable to build SDL2 test programs" ${SDL2COMPAT_MAINPROJECT})
option(SDL2COMPAT_INSTALL "Enable installing SDL2-compat" ${SDL2COMPAT_MAINPROJECT})
cmake_dependent_option(SDL2COMPAT_INSTALL_CPACK "Create binary SDL2_compat archive using CPack" ${SDL2COMPAT_MAINPROJECT} "SDL2COMPAT_INSTALL" OFF)
cmake_dependent_option(SDL2COMPAT_INSTALL_TESTS "Install test-cases" OFF "SDL2COMPAT_INSTALL;SDL2COMPAT_TESTS" OFF)
cmake_dependent_option(SDL2COMPAT_INSTALL_SDL3 "Install SDL3 next to SDL2" OFF "WIN32;SDL2COMPAT_INSTALL" OFF)
option(SDL2COMPAT_ASAN "Use AddressSanitizer to detect memory errors" OFF)
option(SDL2COMPAT_STATIC "Enable building static SDL2 link library" OFF)
option(SDL2COMPAT_WERROR "Treat warnings as errors" OFF)
set(SDL2COMPAT_VENDOR_INFO "" CACHE STRING "Vendor name and/or version to add to SDL_REVISION")
set(SDL2COMPAT_FRAMEWORK FALSE)
#FIXME: should SDL2-compat modify vendor_info/revision information?

if(SDL2COMPAT_STATIC AND NOT (CMAKE_SYSTEM_NAME MATCHES "Linux"))
  message(FATAL_ERROR "Static builds are only supported on Linux.")
endif()

macro(check_add_debug_flag FLAG SUFFIX)
  check_c_compiler_flag(${FLAG} HAS_C_FLAG_${SUFFIX})
  if(HAS_C_FLAG_${SUFFIX})
    string(APPEND CMAKE_C_FLAGS_DEBUG " ${FLAG}")
  endif()
endmacro()

macro(asan_check_add_debug_flag ASAN_FLAG)
  check_add_debug_flag("-fsanitize=${ASAN_FLAG}" "${ASAN_FLAG}")
  if(HAS_C_${ASAN_FLAG} OR HAS_CXX_${ASAN_FLAG})
    set(HAVE_ASAN ON)
  endif()
endmacro()

macro(asan_check_add_debug_flag2 ASAN_FLAG)
  # for some sanitize flags we have to manipulate the CMAKE_REQUIRED_LIBRARIES:
  # http://cmake.3232098.n2.nabble.com/CHECK-CXX-COMPILER-FLAG-doesn-t-give-correct-result-for-fsanitize-address-tp7600216p7600217.html

  set(FLAG "-fsanitize=${ASAN_FLAG}")

  cmake_push_check_state()
  list(APPEND CMAKE_REQUIRED_LIBRARIES ${FLAG} asan)

  check_c_compiler_flag (${FLAG} HAS_C_FLAG_${ASAN_FLAG})
  if (HAS_C_FLAG_${ASAN_FLAG})
    string(APPEND CMAKE_C_FLAGS_DEBUG " ${FLAG}")
  endif()

  cmake_pop_check_state()
  if(HAS_C_${ASAN_FLAG} OR HAS_CXX_${ASAN_FLAG})
    set(HAVE_ASAN ON)
  endif()
endmacro()

# enable AddressSanitizer if supported
if(SDL2COMPAT_ASAN)
  asan_check_add_debug_flag2("address")
  asan_check_add_debug_flag("bool")
  asan_check_add_debug_flag("bounds")
  asan_check_add_debug_flag("enum")
  asan_check_add_debug_flag("float-cast-overflow")
  asan_check_add_debug_flag("float-divide-by-zero")
  asan_check_add_debug_flag("nonnull-attribute")
  asan_check_add_debug_flag("returns-nonnull-attribute")
  asan_check_add_debug_flag("signed-integer-overflow")
  asan_check_add_debug_flag("undefined")
  asan_check_add_debug_flag("vla-bound")
  asan_check_add_debug_flag("leak")
  # The object size sanitizer has no effect on unoptimized builds on Clang,
  # but causes warnings.
  if(NOT USE_CLANG OR CMAKE_BUILD_TYPE STREQUAL "")
    asan_check_add_debug_flag("object-size")
  endif()
endif()

SDL_DetectCompiler()
SDL_DetectCPUArchitecture()
SDL_DetectCMakePlatform()
CheckOpenGLES()

if(NOT MSVC AND NOT APPLE AND NOT OPENBSD)
  cmake_push_check_state()
  set(CMAKE_REQUIRED_FLAGS "-Wl,--no-undefined")
  check_c_compiler_flag("" HAVE_NO_UNDEFINED)
  cmake_pop_check_state()
  if(HAVE_NO_UNDEFINED)
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,--no-undefined")
    set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,--no-undefined")
  endif()
endif()

if(NOT TARGET SDL3::Headers)
  find_package(SDL3 QUIET COMPONENTS Headers)
endif()

if(NOT TARGET SDL3::Headers)
  find_path(SDL3_INCLUDE_DIRS
    NAMES "SDL3/SDL.h"
  )
  if(NOT SDL3_INCLUDE_DIRS)
    message(FATAL_ERROR "Cannot find SDL3 headers: configure with -DCMAKE_PREFIX_PATH=\"/root/to/SDL3\" or -DSDL3_INCLUDE_DIRS=\"/root/to/SDL3/include\"")
  endif()
  add_library(SDL3::Headers INTERFACE IMPORTED)
  set_property(TARGET SDL3::Headers PROPERTY INTERFACE_INCLUDE_DIRECTORIES "${SDL3_INCLUDE_DIRS}")
endif()

if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/REVISION.txt")
  file(STRINGS "${CMAKE_CURRENT_SOURCE_DIR}/REVISION.txt" REVISION_txt_lines)
  list(GET REVISION_txt_lines 0 SDL_REVISION)
else()
  find_package(Git)
  if(Git_FOUND)
    execute_process(
      COMMAND "${GIT_EXECUTABLE}" describe --always --tags --long
      WORKING_DIRECTORY "${PROJECT_SOURCE_DIR}"
      RESULT_VARIABLE GIT_REVISION_STATUS
      OUTPUT_VARIABLE GIT_REVISION
      ERROR_QUIET OUTPUT_STRIP_TRAILING_WHITESPACE)
  else()
    set(GIT_REVISION_STATUS 1)
    set(GIT_REVISION "")
  endif()

  if(GIT_REVISION_STATUS EQUAL 0)
    if(GIT_REVISION MATCHES "^[0-9a-f]+$")
      # Just a truncated sha1, so prefix it with the version number
      set(SDL_REVISION "SDL-${SDL_VERSION}-g${GIT_REVISION}")
    else()
      # e.g. release-2.24.0-542-g96361fc47
      set(SDL_REVISION "SDL-${GIT_REVISION}")
    endif()
  else()
    set(SDL_REVISION "SDL-${SDL_VERSION}-no-vcs")
  endif()
endif()

if(SDL2COMPAT_VENDOR_INFO)
  set(SDL2COMPAT_REVISION "${SDL_REVISION} (${SDL2COMPAT_VENDOR_INFO})")
else()
  set(SDL2COMPAT_REVISION "${SDL_REVISION}")
endif()
configure_file(include/SDL2/SDL_revision.h.cmake include/SDL2/SDL_revision.h @ONLY)

set(SDL2COMPAT_SRCS
  src/sdl2_compat.c
  src/dynapi/SDL_dynapi.c
)

if(UNIX AND NOT ANDROID)
  find_package(X11 REQUIRED)
  include_directories(${X11_INCLUDE_DIR})
endif()

if(APPLE)
  list(APPEND SDL2COMPAT_SRCS
    "src/sdl2_compat_objc.m"
  )

  if(POLICY CMP0068)
    cmake_policy(SET CMP0068 NEW)  # on macOS, don't let RPATH affect install_name.
  endif()
endif()

if(WIN32)
  list(APPEND SDL2COMPAT_SRCS
    "src/sdl2_mslibc.c"
    "src/version.rc"
  )
endif()
if(MSVC)
  if(SDL_CPU_X64)
    list(APPEND SDL2COMPAT_SRCS
      "src/sdl2_mslibc_x64.masm"
    )
    enable_language(ASM_MASM)
    set_property(SOURCE "src/sdl2_mslibc_x64.masm" PROPERTY LANGUAGE "ASM_MASM")
  elseif(SDL_CPU_ARM64)
    list(APPEND SDL2COMPAT_SRCS
      "src/sdl2_mslibc_arm64.masm"
    )
    enable_language(ASM_MARMASM)
    set_property(SOURCE "src/sdl2_mslibc_arm64.masm" PROPERTY LANGUAGE "ASM_MARMASM")
  elseif(SDL_CPU_ARM32)
    # FIXME: Add ARM32 _chkstk implementation
  endif()
endif()

if(APPLE)
  foreach(SOURCE_FILE ${SDL2COMPAT_SRCS})
    get_filename_component(FILE_EXTENSION ${SOURCE_FILE} EXT)
    if(FILE_EXTENSION STREQUAL "m")
      set_property(SOURCE ${SOURCE_FILE} APPEND_STRING PROPERTY COMPILE_FLAGS " -x objective-c")
    endif()
  endforeach()
endif()

add_library(SDL2 SHARED ${SDL2COMPAT_SRCS})
add_library(SDL2::SDL2 ALIAS SDL2)
SDL_AddCommonCompilerFlags(SDL2 WERROR "${SDL2COMPAT_WERROR}")
target_link_libraries(SDL2 PRIVATE $<BUILD_INTERFACE:SDL3::Headers>)
target_include_directories(SDL2
  INTERFACE
    "$<BUILD_INTERFACE:${PROJECT_BINARY_DIR}/include>"
    "$<BUILD_INTERFACE:${PROJECT_BINARY_DIR}/include/SDL2>"
    "$<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>"
    "$<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include/SDL2>"
    "$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>"
    "$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/SDL2>"
)
target_compile_definitions(SDL2 PRIVATE "SDL2COMPAT_REVISION=\"${SDL2COMPAT_REVISION}\"")
if(TARGET SDL3::SDL3-shared)
  set_property(TARGET SDL2 PROPERTY BUILD_RPATH $<TARGET_FILE_DIR:SDL3::SDL3-shared>)
endif()

if(MSVC AND SDL_CPU_X64)
  target_compile_options(SDL2 PRIVATE "$<$<COMPILE_LANGUAGE:ASM_MASM>:/nologo>")
endif()
set(EXTRA_CFLAGS )
if(CMAKE_C_COMPILER_ID MATCHES "Clang|GNU")
  set(EXTRA_CFLAGS "${EXTRA_CFLAGS} -Wall")
  check_c_compiler_flag(-fvisibility=hidden HAVE_FVISIBILITY_HIDDEN)
  if(HAVE_FVISIBILITY_HIDDEN AND (APPLE OR UNIX))
    set(EXTRA_CFLAGS "${EXTRA_CFLAGS} -fvisibility=hidden")
  endif()
  check_c_compiler_flag(-Wdeclaration-after-statement HAVE_WDECLARATION_AFTER_STATEMENT)
  if(HAVE_WDECLARATION_AFTER_STATEMENT)
    set(EXTRA_CFLAGS "${EXTRA_CFLAGS} -Wdeclaration-after-statement")
  endif()
  if(SDL2COMPAT_WERROR)
    check_c_compiler_flag(-Werror=declaration-after-statement HAVE_WERROR_DECLARATION_AFTER_STATEMENT)
    if(HAVE_WERROR_DECLARATION_AFTER_STATEMENT)
      set(EXTRA_CFLAGS "${EXTRA_CFLAGS} -Werror=declaration-after-statement")
    endif()
  endif()
endif()

check_include_file("winapifamily.h" HAVE_WINAPIFAMILY_H)
if(HAVE_WINAPIFAMILY_H)
  set(EXTRA_CFLAGS "${EXTRA_CFLAGS} -DSDL_HAVE_WINAPIFAMILY_H")
endif()

# This isn't needed for compat libSDL2 who use SDL3 headers.
# SDL2_test and SDL2_main (and the tes programs) still need
# this, because they use SDL2 headers.
check_include_file("immintrin.h" HAVE_IMMINTRIN_H)
if(NOT HAVE_IMMINTRIN_H)
  set(EXTRA_CFLAGS "${EXTRA_CFLAGS} -DSDL_DISABLE_IMMINTRIN_H")
endif()
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${EXTRA_CFLAGS}")
string(STRIP "${CMAKE_C_FLAGS}" CMAKE_C_FLAGS)

if(UNIX AND NOT APPLE)
  target_compile_definitions(SDL2 PRIVATE "_REENTRANT")
  target_link_libraries(SDL2 PRIVATE ${CMAKE_DL_LIBS})
endif()
if(ANDROID)
  target_link_libraries(SDL2 PRIVATE log)
endif()
if(APPLE)
  # !!! FIXME: check OUTPUT_NAME value.
  set_target_properties(SDL2 PROPERTIES COMPILE_DEFINITIONS "_THREAD_SAFE")
  target_link_libraries(SDL2 PRIVATE
    "-Wl,-compatibility_version,${SDL_DYLIB_COMPAT_VERSION}"
    "-Wl,-current_version,${SDL_DYLIB_CURRENT_VERSION}"
    "-Wl,-framework,AppKit"
  )
  set_target_properties(SDL2 PROPERTIES
      OUTPUT_NAME "SDL2-2.0"
      SOVERSION "0"
  )
elseif(UNIX AND NOT ANDROID)
  # !!! FIXME: check OUTPUT_NAME value
  set_target_properties(SDL2 PROPERTIES
    OUTPUT_NAME "SDL2-2.0"
    VERSION "${SDL_SO_VERSION}"
    SOVERSION "${SDL_SO_VERSION_MAJOR}"
  )
elseif(WIN32)
  set_target_properties(SDL2 PROPERTIES DEFINE_SYMBOL "DLL_EXPORT")
  # avoid DLL having 'lib' prefix with MinGW
  set_target_properties(SDL2 PROPERTIES
    OUTPUT_NAME "SDL2"
    PREFIX ""
  )
else()
  set_target_properties(SDL2 PROPERTIES
    VERSION "${PROJECT_VERSION}"
    SOVERSION "0"
    OUTPUT_NAME "SDL2"
  )
endif()

if(MINGW)
  set_property(TARGET SDL2 APPEND_STRING PROPERTY LINK_FLAGS " -nostdlib")
  if(USE_CLANG)
    if(SDL_CPU_X86)
      target_link_libraries(SDL2 PRIVATE clang_rt.builtins-i386)
    endif()
    if(SDL_CPU_X64)
      target_link_libraries(SDL2 PRIVATE clang_rt.builtins-x86_64)
    endif()
  else()
    # need libgcc for emulating 64-bit operations (e.g. 64-bit division) or __chkstk_ms()
    target_link_libraries(SDL2 PRIVATE gcc)
    set_property(TARGET SDL2 APPEND_STRING PROPERTY LINK_FALGS " -static-libgcc")
  endif()
endif()
if(MSVC)
  # Don't try to link with the default set of libraries.
  target_compile_options(SDL2 PRIVATE "$<$<COMPILE_LANGUAGE:C>:/GS->")
  if(SDL_CPU_X86)  # don't emit SSE2 in x86 builds
    target_compile_options(SDL2 PRIVATE "$<$<COMPILE_LANGUAGE:C>:/arch:SSE>")
  endif()
  if(NOT MSVC_CLANG AND NOT SDL_CPU_ARM32)
    set_property(TARGET SDL2 APPEND_STRING PROPERTY LINK_FLAGS " /NODEFAULTLIB")
  endif()
  # Make sure /RTC1 is disabled: (from SDL2 CMake)
  foreach(flag_var
    CMAKE_C_FLAGS CMAKE_C_FLAGS_DEBUG CMAKE_C_FLAGS_RELEASE
    CMAKE_C_FLAGS_MINSIZEREL CMAKE_C_FLAGS_RELWITHDEBINFO)
    string(REGEX REPLACE "/RTC(su|[1su])" "" ${flag_var} "${${flag_var}}")
  endforeach(flag_var)
endif()

# !!! FIXME: what needs to be done for sdl2-compat? Look into this.
## SDL2main library...
if(WIN32)
    add_library(SDL2main STATIC src/SDLmain/windows/SDL_windows_main.c)
    target_link_libraries(SDL2main PRIVATE shell32)
else()
    add_library(SDL2main STATIC src/SDLmain/dummy/SDL_dummy_main.c)
endif()
add_library(SDL2::SDL2main ALIAS SDL2main)
target_include_directories(SDL2main
    PRIVATE
      "$<BUILD_INTERFACE:${PROJECT_BINARY_DIR}/include>"
      "$<BUILD_INTERFACE:${PROJECT_BINARY_DIR}/include/SDL2>"
      "$<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>"
      "$<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include/SDL2>"
)
if(MINGW OR CYGWIN)
  if(CMAKE_SIZEOF_VOID_P EQUAL 4)
    target_link_libraries(SDL2main PUBLIC "$<$<STREQUAL:$<TARGET_PROPERTY:TYPE>,EXECUTABLE>:-Wl,--undefined=_WinMain@16>")
  else()
    target_link_libraries(SDL2main PUBLIC "$<$<STREQUAL:$<TARGET_PROPERTY:TYPE>,EXECUTABLE>:-Wl,--undefined=WinMain>")
  endif()
endif()

# SDLtest library...
add_library(SDL2_test STATIC
  src/test/SDL_test_assert.c
  src/test/SDL_test_common.c
  src/test/SDL_test_compare.c
  src/test/SDL_test_crc32.c
  src/test/SDL_test_font.c
  src/test/SDL_test_fuzzer.c
  src/test/SDL_test_harness.c
  src/test/SDL_test_imageBlit.c
  src/test/SDL_test_imageBlitBlend.c
  src/test/SDL_test_imageFace.c
  src/test/SDL_test_imagePrimitives.c
  src/test/SDL_test_imagePrimitivesBlend.c
  src/test/SDL_test_log.c
  src/test/SDL_test_md5.c
  src/test/SDL_test_memory.c
  src/test/SDL_test_random.c
)
add_library(SDL2::SDL2test ALIAS SDL2_test)
target_include_directories(SDL2_test
  PUBLIC
    "$<BUILD_INTERFACE:${PROJECT_BINARY_DIR}/include>"
    "$<BUILD_INTERFACE:${PROJECT_BINARY_DIR}/include/SDL2>"
    "$<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>"
    "$<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include/SDL2>"
    "$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>"
    "$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/SDL2>"
)

if(SDL2COMPAT_TESTS)
  enable_testing()
  add_subdirectory(test)
endif()

set(installed_targets SDL2)

if(SDL2COMPAT_STATIC)
  add_library(SDL2-static STATIC ${SDL2COMPAT_SRCS})
  add_library(SDL2::SDL2-static ALIAS SDL2-static)
  SDL_AddCommonCompilerFlags(SDL2-static WERROR "${SDL2COMPAT_WERROR}")
  target_link_libraries(SDL2-static PRIVATE $<BUILD_INTERFACE:SDL3::Headers>)
  target_include_directories(SDL2-static
    INTERFACE
      "$<BUILD_INTERFACE:${PROJECT_BINARY_DIR}/include>"
      "$<BUILD_INTERFACE:${PROJECT_BINARY_DIR}/include/SDL2>"
      "$<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>"
      "$<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include/SDL2>"
      "$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>"
      "$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/SDL2>"
  )
  target_compile_definitions(SDL2-static PRIVATE _REENTRANT)
  target_compile_definitions(SDL2-static PRIVATE "SDL2COMPAT_REVISION=\"${SDL2COMPAT_REVISION}\"")
  target_link_libraries(SDL2-static PRIVATE ${CMAKE_DL_LIBS})
  set_target_properties(SDL2-static PROPERTIES
    VERSION "${PROJECT_VERSION}"
    OUTPUT_NAME "SDL2"
  )
  if(CMAKE_STATIC_LIBRARY_PREFIX STREQUAL "" AND CMAKE_STATIC_LIBRARY_SUFFIX STREQUAL ".lib")
    set_target_properties(SDL2-static PROPERTIES
      OUTPUT_NAME "SDL2-static"
    )
  endif()

  list(APPEND installed_targets SDL2-static)
endif()

if(SDL2COMPAT_INSTALL)
  if(WIN32 AND NOT MINGW)
    set(SDL2COMPAT_INSTALL_CMAKEDIR_DEFAULT "cmake")
    set(LICENSES_PREFIX "licenses/SDL2")
  else()
    set(SDL2COMPAT_INSTALL_CMAKEDIR_DEFAULT "${CMAKE_INSTALL_LIBDIR}/cmake/SDL2")
    set(LICENSES_PREFIX "${CMAKE_INSTALL_DATAROOTDIR}/licenses/SDL2")
  endif()
  set(SDL2COMPAT_INSTALL_CMAKEDIR "${SDL2COMPAT_INSTALL_CMAKEDIR_DEFAULT}" CACHE STRING "Location where to install SDL2Config.cmake")

  list(APPEND installed_targets SDL2main SDL2_test)
  foreach(target ${installed_targets})
    install(TARGETS ${target} EXPORT ${target}Targets
      ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
      LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
      RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}"
    )
    install(EXPORT ${target}Targets
      FILE ${target}Targets.cmake
      NAMESPACE SDL2::
      DESTINATION "${SDL2COMPAT_INSTALL_CMAKEDIR}"
    )
  endforeach()
  install(FILES "${PROJECT_SOURCE_DIR}/LICENSE.txt"
    DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/licenses/sdl2-compat"
  )
  if(SDL2COMPAT_INSTALL_SDL3 AND TARGET SDL3::SDL3-shared)
    if(WIN32)
      install(FILES $<TARGET_FILE:SDL3::SDL3-shared>
        DESTINATION "${CMAKE_INSTALL_BINDIR}"
      )
    else()
      message(WARNING "SDL2COMPAT_INSTALL_SDL3 not implemented for current platform")
    endif()
  endif()
  if(MSVC)
    SDL_install_pdb(SDL2 "${CMAKE_INSTALL_BINDIR}")
    SDL_install_pdb(SDL2main "${CMAKE_INSTALL_LIBDIR}")
    SDL_install_pdb(SDL2_test "${CMAKE_INSTALL_LIBDIR}")
  endif()
  configure_package_config_file(SDL2Config.cmake.in "${CMAKE_CURRENT_BINARY_DIR}/SDL2Config.cmake"
    PATH_VARS CMAKE_INSTALL_PREFIX CMAKE_INSTALL_FULL_BINDIR CMAKE_INSTALL_FULL_INCLUDEDIR CMAKE_INSTALL_FULL_LIBDIR
    INSTALL_DESTINATION "${SDL2COMPAT_INSTALL_CMAKEDIR}"
  )
  write_basic_package_version_file("${CMAKE_CURRENT_BINARY_DIR}/SDL2ConfigVersion.cmake"
    VERSION ${SDL_VERSION}
    COMPATIBILITY AnyNewerVersion
  )
  install(
    FILES
      cmake/sdl2-config.cmake
      cmake/sdl2-config-version.cmake
      ${CMAKE_CURRENT_BINARY_DIR}/SDL2Config.cmake
      ${CMAKE_CURRENT_BINARY_DIR}/SDL2ConfigVersion.cmake
    DESTINATION "${SDL2COMPAT_INSTALL_CMAKEDIR}"
  )

  file(GLOB SDL2_INCLUDE_FILES ${PROJECT_SOURCE_DIR}/include/SDL2/*.h)
  set(SDL2_COPIED_INCLUDE_FILES)
  foreach(_hdr IN LISTS SDL2_INCLUDE_FILES)
    if(_hdr MATCHES ".*(SDL_config).*")
      # FIXME: how to handle SDL_config.h? Generate a new one from SDL_config.h.cmake? Copy SDL_config.h + SDL_config_xxxxx.h?
      install(FILES "${_hdr}" DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/SDL2")
    elseif(_hdr MATCHES ".*(SDL_revision).*")
      list(REMOVE_ITEM SDL2_INCLUDE_FILES "${_hdr}")
    else()
      install(FILES "${_hdr}" DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/SDL2")
    endif()
  endforeach()
  install(FILES "${PROJECT_BINARY_DIR}/include/SDL2/SDL_revision.h" DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/SDL2")

  if(MSVC)
    set(SDL_CFLAGS "")
    set(SDL_RLD_FLAGS "")
    set(SDL_LIBS "-lSDL2main -lSDL2")
    set(SDL_STATIC_LIBS "-lshell32")
  elseif(MINGW)
    set(SDL_CFLAGS "")
    set(SDL_RLD_FLAGS "")
    set(SDL_LIBS "-lmingw32 -lSDL2main -lSDL2 -mwindows")
    set(SDL_STATIC_LIBS "-lshell32")
  elseif(APPLE)
    set(SDL_CFLAGS "-D_THREAD_SAFE")
    set(SDL_RLD_FLAGS "")  # !!! FIXME: this forces rpath, which we might want?
    set(SDL_LIBS "-lSDL2main -lSDL2 -Wl,-framework,Cocoa")
    set(SDL_STATIC_LIBS "")
  else() # unix
    set(SDL_CFLAGS "-D_GNU_SOURCE=1 -D_REENTRANT")
    set(SDL_RLD_FLAGS "")  # !!! FIXME: this forces rpath, which we might want?
    set(SDL_LIBS "-lSDL2")
    set(SDL_STATIC_LIBS "")
    foreach(lib ${CMAKE_DL_LIBS})
      set(SDL_STATIC_LIBS "-l${lib}")
    endforeach()
    if(NOT SDL2COMPAT_STATIC)
      set(SDL_STATIC_LIBS "")
    endif()
  endif()

  # !!! FIXME: do we _want_ static builds?
  if(SDL2COMPAT_STATIC)
    set(ENABLE_STATIC_TRUE "")
    set(ENABLE_STATIC_FALSE "#")
  else()
    set(ENABLE_STATIC_TRUE "#")
    set(ENABLE_STATIC_FALSE "")
  endif()
  set(ENABLE_SHARED_TRUE "")
  set(ENABLE_SHARED_FALSE "#")

  configure_file(sdl2-compat.pc.in sdl2-compat.pc @ONLY)
  install(FILES "${CMAKE_CURRENT_BINARY_DIR}/sdl2-compat.pc"
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig
  )

  configure_file(sdl2-config.in "${CMAKE_CURRENT_BINARY_DIR}/sdl2-config" @ONLY)
  install(PROGRAMS "${CMAKE_CURRENT_BINARY_DIR}/sdl2-config" DESTINATION "${CMAKE_INSTALL_BINDIR}")

  # uninstall
  if(NOT TARGET uninstall)
    configure_file(cmake/cmake_uninstall.cmake.in "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake" IMMEDIATE @ONLY)
    add_custom_target(uninstall
      COMMAND ${CMAKE_COMMAND} -P "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake")
  endif()

  set(SOEXT ${CMAKE_SHARED_LIBRARY_SUFFIX})
  get_target_property(SONAME SDL2 OUTPUT_NAME)
  if(UNIX AND NOT ANDROID)
    install(CODE "
      execute_process(COMMAND \"${CMAKE_COMMAND}\" -E create_symlink
        \"lib${SONAME}${SOPOSTFIX}${SOEXT}\" \"libSDL2${SOPOSTFIX}${SOEXT}\"
        WORKING_DIRECTORY \"${CMAKE_CURRENT_BINARY_DIR}\")")
    install(FILES ${CMAKE_CURRENT_BINARY_DIR}/libSDL2${SOPOSTFIX}${SOEXT} DESTINATION "${CMAKE_INSTALL_LIBDIR}")
  endif()

  install(FILES "${PROJECT_SOURCE_DIR}/sdl2.m4" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/aclocal")

  if(SDL2COMPAT_INSTALL_CPACK)
    if(SDL2COMPAT_FRAMEWORK)
      set(CPACK_GENERATOR "DragNDrop")
    elseif(MSVC)
      set(CPACK_GENERATOR "ZIP")
    else()
      set(CPACK_GENERATOR "TGZ")
    endif()
    configure_file(cmake/CPackProjectConfig.cmake.in CPackProjectConfig.cmake @ONLY)
    set(CPACK_PROJECT_CONFIG_FILE "${PROJECT_BINARY_DIR}/CPackProjectConfig.cmake")
    # CPACK_SOURCE_PACKAGE_FILE_NAME must end with "-src" (so we can block creating a source archive)
    set(CPACK_SOURCE_PACKAGE_FILE_NAME "SDL${PROJECT_VERSION_MAJOR}-${PROJECT_VERSION}-src")
    set(CPACK_PACKAGE_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}/dist")
    include(CPack)
  endif()
endif()
