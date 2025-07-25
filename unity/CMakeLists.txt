# Copyright 2017-2023 Valve Corporation.
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

cmake_minimum_required(VERSION 3.17)

project(Phonon VERSION 4.6.1)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_MODULE_PATH "${CMAKE_HOME_DIRECTORY}/build")


#
# SYSTEM INTROSPECTION
#

# OS detection
if (WIN32)
    set(IPL_OS_WINDOWS TRUE)
elseif (UNIX AND CMAKE_SYSTEM_NAME STREQUAL "Linux")
    set(IPL_OS_LINUX TRUE)
elseif (APPLE AND CMAKE_SYSTEM_NAME STREQUAL "Darwin")
    set(IPL_OS_MACOS TRUE)
elseif (ANDROID)
    set(IPL_OS_ANDROID TRUE)
elseif (APPLE AND CMAKE_SYSTEM_NAME STREQUAL "iOS")
    set(IPL_OS_IOS TRUE)
elseif (CMAKE_SYSTEM_NAME STREQUAL "Emscripten")
    set(IPL_OS_WASM TRUE)
else()
    set(IPL_OS_UNSUPPORTED TRUE)
endif()

# CPU architecture detection
if (IPL_OS_WINDOWS OR IPL_OS_LINUX)
    if (CMAKE_SIZEOF_VOID_P EQUAL 8)
        set(IPL_CPU_X64 TRUE)
    else()
        set(IPL_CPU_X86 TRUE)
    endif()
elseif (IPL_OS_MACOS)
elseif (IPL_OS_ANDROID)
    if (CMAKE_ANDROID_ARCH STREQUAL "arm")
        set(IPL_CPU_ARMV7 TRUE)
    elseif (CMAKE_ANDROID_ARCH STREQUAL "arm64")
        set(IPL_CPU_ARMV8 TRUE)
    elseif (CMAKE_ANDROID_ARCH STREQUAL "x86")
        set(IPL_CPU_X86 TRUE)
    elseif (CMAKE_ANDROID_ARCH STREQUAL "x86_64")
        set(IPL_CPU_X64 TRUE)
    endif()
elseif (IPL_OS_IOS)
    set(IPL_CPU_ARMV8 TRUE)
elseif (IPL_OS_WASM)
    set(IPL_CPU_ARMV7 TRUE)
endif()


#
# OPTIONS
#

# Whether to compile a static or dynamic library
option(BUILD_SHARED_LIBS "Build shared libraries instead of static." ON)

# Options for all platforms
option(STEAMAUDIOUNITY_BUILD_DOCS "Build documentation." OFF)

# Paths for find_package
set(Sphinx_EXECUTABLE_DIR "" CACHE PATH "Directory containing the Sphinx binary.")
set(Unity_EXECUTABLE_DIR "" CACHE PATH "Directory containing the Unity binary.")
mark_as_advanced(Sphinx_EXECUTABLE_DIR Unity_EXECUTABLE_DIR)


#
# COMPILER FLAGS
#

# Flags determined by system introspection
if (IPL_OS_WINDOWS)
    add_definitions(-DIPL_OS_WINDOWS)
endif()
if (IPL_OS_LINUX)
    add_definitions(-DIPL_OS_LINUX)
endif()
if (IPL_OS_MACOS)
    add_definitions(-DIPL_OS_MACOSX)
endif()
if (IPL_OS_ANDROID)
    add_definitions(-DIPL_OS_ANDROID)
endif()
if (IPL_OS_IOS)
    add_definitions(-DIPL_OS_IOS)
endif()
if (IPL_OS_WASM)
    add_definitions(-DIPL_OS_WASM)
endif()
if (IPL_OS_UNSUPPORTED)
    add_definitions(-DIPL_OS_UNSUPPORTED)
endif()

if (IPL_CPU_X86)
    add_definitions(-DIPL_CPU_X86)
endif()
if (IPL_CPU_X64)
    add_definitions(-DIPL_CPU_X64)
endif()
if (IPL_CPU_ARMV7)
    add_definitions(-DIPL_CPU_ARMV7)
endif()
if (IPL_CPU_ARMV8)
    add_definitions(-DIPL_CPU_ARM64)
endif()

# Windows flags
if (IPL_OS_WINDOWS)
    add_compile_definitions(
        WIN32
        WINDOWS
        UNICODE
        _UNICODE
        _CRT_SECURE_NO_WARNINGS
        WIN32_LEAN_AND_MEAN
        NOMINMAX
        _DISABLE_EXTENDED_ALIGNED_STORAGE
        $<IF:$<CONFIG:Debug>,_DEBUG,NDEBUG>
    )
    add_compile_options(
        /EHsc /MP /W3 /Zi
        $<IF:$<CONFIG:Debug>,/MTd,/MT>
        $<$<CONFIG:Debug>:/RTC1>
        $<$<NOT:$<CONFIG:Debug>>:/Ox>
        $<$<CONFIG:Release>:/GL>
    )
    add_link_options(
        /DEBUG
        $<$<CONFIG:Release>:/OPT:REF>
        $<$<CONFIG:Release>:/OPT:ICF>
        $<$<CONFIG:Release>:/LTCG>
        $<$<CONFIG:Release>:/MANIFEST:NO>
        $<IF:$<CONFIG:Release>,/INCREMENTAL:NO,/INCREMENTAL>
    )
endif()

# Linux flags
if (IPL_OS_LINUX)
    add_compile_options(-fPIC -g -Wl,--no-undefined)
    if (IPL_CPU_X86)
        add_compile_options(-m32 -mfpmath=sse -march=native)
    endif()
endif()

# macOS flags
if (IPL_OS_MACOS)
    set(CMAKE_OSX_ARCHITECTURES "x86_64;arm64")
    set(CMAKE_OSX_DEPLOYMENT_TARGET "10.13")
    add_compile_options(-Wno-extern-c-compat) # Suppress warning about empty API structs.
endif()

# Android flags
if (IPL_OS_ANDROID)
    add_compile_options(-fPIC -g $<$<CONFIG:Release>:-O3>)
    add_link_options(-static-libstdc++)

    # Enable 16 KB page size support. (https://developer.android.com/about/versions/15/behavior-changes-all#16-kb)
    add_link_options("-Wl,-z,max-page-size=16384")
endif()

# iOS flags
# todo

# WASM flags
if (IPL_OS_WASM)
    add_compile_options(-msimd128 -mfpu=neon)
    add_link_options(-sALLOW_MEMORY_GROWTH=1 -sSINGLE_FILE=1)
endif()


#
# DEPENDENCIES
#

find_package(Unity REQUIRED)

find_package(SteamAudio REQUIRED)

if (STEAMAUDIOUNITY_BUILD_DOCS)
    find_package(Sphinx)
    if (NOT Sphinx_FOUND)
        message(STATUS "Disabling docs")
        set(STEAMAUDIOUNITY_BUILD_DOCS OFF)
    endif()
endif()


#
# TARGETS
#

add_subdirectory(src/native)

if (STEAMAUDIOUNITY_BUILD_DOCS)
    add_subdirectory(doc)
endif()


#
# PACKAGING
#

install(FILES ${CMAKE_HOME_DIRECTORY}/../core/THIRDPARTY.md DESTINATION root)

set(CPACK_PACKAGE_VENDOR "Valve Corporation")
set(CPACK_PACKAGE_FILE_NAME steamaudio_unity)
set(CPACK_PACKAGE_DIRECTORY ${CMAKE_HOME_DIRECTORY}/dist)
set(CPACK_INSTALL_CMAKE_PROJECTS "")
set(CPACK_EXTERNAL_PACKAGE_SCRIPT ${CMAKE_HOME_DIRECTORY}/build/package_unity.cmake)
set(CPACK_INSTALLED_DIRECTORIES
    ${CMAKE_INSTALL_PREFIX}/unity unity
    ${CMAKE_INSTALL_PREFIX}/symbols symbols
    ${CMAKE_INSTALL_PREFIX}/doc doc
    ${CMAKE_INSTALL_PREFIX}/root .
)

include(CPack)
