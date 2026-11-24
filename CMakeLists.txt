cmake_minimum_required(VERSION 3.12)
project(backtrace_crashpad LANGUAGES C CXX)
cmake_policy(SET CMP0077 NEW)

# C and C++ only: ml64 (MASM) rejects /Z7.
if (MSVC)
    add_compile_options(
        "$<$<AND:$<CONFIG:Release>,$<OR:$<COMPILE_LANGUAGE:C>,$<COMPILE_LANGUAGE:CXX>>>:/Z7>")
    add_link_options(
        "$<$<CONFIG:Release>:/DEBUG>"
        "$<$<CONFIG:Release>:/OPT:REF>"
        "$<$<CONFIG:Release>:/OPT:ICF>")
endif ()

if (UNIX AND NOT APPLE)
    set(LINUX TRUE)
endif()

if (WIN32)
    enable_language(ASM_MASM)
else ()
    enable_language(ASM)
endif ()

if (APPLE)
    find_package(Python3 COMPONENTS Interpreter REQUIRED)
endif()

if (LINUX)
    find_package(Threads REQUIRED)
endif (LINUX)

find_package(ZLIB)

add_library(backtrace_common INTERFACE)

target_compile_definitions(backtrace_common INTERFACE
    MINI_CHROMIUM_INTERNAL_BUILDFLAG_VALUE_IS_CHROMEOS_ASH=MINI_CHROMIUM_INTERNAL_BUILDFLAG_VALUE_IS_CHROMEOS
    MINI_CHROMIUM_INTERNAL_BUILDFLAG_VALUE_IS_CHROMEOS_LACROS=MINI_CHROMIUM_INTERNAL_BUILDFLAG_VALUE_IS_CHROMEOS
)

if (NOT ZLIB_FOUND)
    add_subdirectory(third_party/zlib)
    add_library(ZLIB::ZLIB ALIAS zlib)
    target_compile_definitions(backtrace_common INTERFACE
        CRASHPAD_INCLUDES CRASHPAD_ZLIB_SOURCE_EMBEDDED)
else()
    target_compile_definitions(backtrace_common INTERFACE
        CRASHPAD_INCLUDES CRASHPAD_ZLIB_SOURCE_SYSTEM)
endif()

target_compile_definitions(backtrace_common INTERFACE
    CRASHPAD_LSS_SOURCE_EMBEDDED)

if (ANDROID)
    option(CRASHPAD_HTTPS_TRANSPORT "Crashpad HTTPS transport" "OPENSSL")
elseif (APPLE)
    # do nothing
elseif (LINUX)
    find_package(CURL)
else()
    set(OPENSSL_USE_STATIC_LIBS TRUE)
    find_package(OpenSSL)
    find_package(CURL)
endif (ANDROID)

if (WIN32)
    add_subdirectory(third_party/getopt)
endif (WIN32)

add_subdirectory(util)
add_subdirectory(compat)
add_subdirectory(snapshot)
add_subdirectory(minidump)
add_subdirectory(third_party/mini_chromium)
add_subdirectory(client)
add_subdirectory(handler)
add_subdirectory(tools)

if (CRASHPAD_BUILD_EXAMPLES)
    add_subdirectory(examples)
endif (CRASHPAD_BUILD_EXAMPLES)

include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

# 1. Collect core targets
set(CRASHPAD_INSTALL_TARGETS client util compat mini_chromium backtrace_common compat_incl)

if(CRASHPAD_HANDLER_GENERATE_STATIC_LIB)
    list(APPEND CRASHPAD_INSTALL_TARGETS handlerlib tools snapshot minidump)
endif()

if(APPLE AND TARGET mig_output)
    list(APPEND CRASHPAD_INSTALL_TARGETS mig_output)
endif()

# 2. Install libraries and crashpad_handler
install(TARGETS ${CRASHPAD_INSTALL_TARGETS} handler
    EXPORT CrashpadTargets
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)

# 3. Install Crashpad headers (preserving directory layout)
install(DIRECTORY client util compat snapshot
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/crashpad
    FILES_MATCHING PATTERN "*.h"
)

# 4. Install mini_chromium headers
install(DIRECTORY third_party/mini_chromium/mini_chromium/base
                  third_party/mini_chromium/mini_chromium/build
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/crashpad/mini_chromium
    FILES_MATCHING PATTERN "*.h"
)

# 5. Export CMake targets so downstream projects can use find_package(Crashpad)
install(EXPORT CrashpadTargets
    FILE CrashpadTargets.cmake
    NAMESPACE Crashpad::
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/Crashpad
)

file(WRITE "${CMAKE_CURRENT_BINARY_DIR}/CrashpadConfig.cmake"
    "include(\"\${CMAKE_CURRENT_LIST_DIR}/CrashpadTargets.cmake\")\n"
)

install(FILES "${CMAKE_CURRENT_BINARY_DIR}/CrashpadConfig.cmake"
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/Crashpad
)
