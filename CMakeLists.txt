cmake_minimum_required(VERSION 4.0)

project(libblake3
    VERSION 1.8.2
    DESCRIPTION "BLAKE3 cryptographic hash function"
    LANGUAGES C CXX ASM
)

if(WIN32)
    enable_language(RC)
endif()
if(MSVC)
    enable_language(ASM_MASM)
endif()
if(NOT DEFINED CMAKE_BUILD_TYPE AND NOT DEFINED CMAKE_CONFIGURATION_TYPES)
    set(CMAKE_BUILD_TYPE "Release")
endif()
set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS OFF)
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
set(CMAKE_VISIBILITY_INLINES_HIDDEN ON)
set(CMAKE_C_VISIBILITY_PRESET "hidden")
set(CMAKE_CXX_VISIBILITY_PRESET "hidden")
foreach(__lang C CXX)
    if(NOT "x${CMAKE_${__lang}_FLAGS}" STREQUAL "x")
        if(MSVC)
            string(REGEX REPLACE " [/-]W[01234] " " " CMAKE_${__lang}_FLAGS ${CMAKE_${__lang}_FLAGS})
        else()
            string(REGEX REPLACE " -W(all)?(extra)? " " " CMAKE_${__lang}_FLAGS ${CMAKE_${__lang}_FLAGS})
            string(REGEX REPLACE " -[W]?pedantic " " " CMAKE_${__lang}_FLAGS ${CMAKE_${__lang}_FLAGS})
        endif()
    endif()
    string(APPEND CMAKE_${__lang}_FLAGS " -w ")
endforeach()

set(__top_level ON)
if("x${CMAKE_SOURCE_DIR}" STREQUAL "x${PROJECT_SOURCE_DIR}")
    message("BLAKE3: Building as a top-level project.")
else()
    set(__top_level OFF)
    message("BLAKE3: Building as a subproject.")
endif()

option(BLAKE3_BUILD_SHARED "Build shared library edition of BLAKE3." ON)
option(BLAKE3_BUILD_STATIC "Build static library edition of BLAKE3." ON)
option(BLAKE3_INSTALL "Enable installation for BLAKE3." OFF)
if(NOT BLAKE3_BUILD_SHARED AND NOT BLAKE3_BUILD_STATIC)
    message(SEND_ERROR "You MUST build either the shared library or the static library or both of them!")
endif()
set(__targets)
if(BLAKE3_BUILD_SHARED)
    add_library(blake3_shared SHARED)
    add_library(BLAKE3::blake3_shared ALIAS blake3_shared)
    set(__dllname BLAKE3)
    if(NOT MSVC)
        set(__dllname libBLAKE3)
    endif()
    set_target_properties(blake3_shared PROPERTIES OUTPUT_NAME ${__dllname})
    target_compile_definitions(blake3_shared
        PUBLIC BLAKE3_DLL
        PRIVATE BLAKE3_DLL_EXPORTS
    )
    if(WIN32)
        target_sources(blake3_shared PRIVATE blake3.rc)
    endif()
    list(APPEND __targets blake3_shared)
endif()
if(BLAKE3_BUILD_STATIC)
    add_library(blake3_static STATIC)
    add_library(BLAKE3::blake3_static ALIAS blake3_static)
    list(APPEND __targets blake3_static)
endif()
if(BLAKE3_BUILD_SHARED AND NOT BLAKE3_BUILD_STATIC)
    add_library(BLAKE3::blake3 ALIAS blake3_shared)
endif()
if(BLAKE3_BUILD_STATIC AND NOT BLAKE3_BUILD_SHARED)
    add_library(BLAKE3::blake3 ALIAS blake3_static)
endif()

add_library(blake3_asm STATIC)
if(MSVC)
    target_sources(blake3_asm PRIVATE
        BLAKE3/blake3_avx2_x86-64_windows_msvc.asm
        BLAKE3/blake3_avx512_x86-64_windows_msvc.asm
        BLAKE3/blake3_sse2_x86-64_windows_msvc.asm
        BLAKE3/blake3_sse41_x86-64_windows_msvc.asm
    )
else()
    if(WIN32)
        target_sources(blake3_asm PRIVATE
            BLAKE3/blake3_avx2_x86-64_windows_gnu.S
            BLAKE3/blake3_avx512_x86-64_windows_gnu.S
            BLAKE3/blake3_sse2_x86-64_windows_gnu.S
            BLAKE3/blake3_sse41_x86-64_windows_gnu.S
        )
    else()
        target_sources(blake3_asm PRIVATE
            BLAKE3/blake3_avx2_x86-64_unix.S
            BLAKE3/blake3_avx512_x86-64_unix.S
            BLAKE3/blake3_sse2_x86-64_unix.S
            BLAKE3/blake3_sse41_x86-64_unix.S
        )
    endif()
endif()

include(GNUInstallDirs)
foreach(__target IN LISTS __targets)
    target_sources(${__target} PRIVATE
        BLAKE3/blake3.c
        BLAKE3/blake3_dispatch.c
        BLAKE3/blake3_portable.c
    )
    target_include_directories(${__target} PUBLIC
        ${CMAKE_CURRENT_SOURCE_DIR}/BLAKE3
    )
    if(__top_level)
        if(WIN32)
            target_compile_definitions(${__target} PRIVATE
                UNICODE _UNICODE
                NOMINMAX
                _USE_MATH_DEFINES
                WIN32_LEAN_AND_MEAN
            )
        endif()
        if(MSVC)
            target_compile_options(${__target} PRIVATE
                /utf-8 /Zc:__cplusplus
                $<$<NOT:$<CONFIG:Debug>>:/Gw /Gy /Zc:inline>
            )
            target_link_options(${__target} PRIVATE
                $<$<NOT:$<CONFIG:Debug>>:/OPT:REF /OPT:ICF>
            )
        else()
            target_compile_options(${__target} PRIVATE
                $<$<NOT:$<CONFIG:Debug>>:-ffunction-sections -fdata-sections>
            )
            target_link_options(${__target} PRIVATE
                $<$<NOT:$<CONFIG:Debug>>:-Wl,--gc-sections>
            )
        endif()
    endif()
    set_target_properties(${__target} PROPERTIES
        VERSION ${PROJECT_VERSION}
    )
    set(__dep_visibility PRIVATE)
    get_target_property(__type ${__target} TYPE)
    if("x${__type}" STREQUAL "xSTATIC_LIBRARY")
        set(__dep_visibility PUBLIC)
    else()
        set_target_properties(${__target} PROPERTIES
            SOVERSION ${PROJECT_VERSION_MAJOR}
        )
    endif()
    target_link_libraries(${__target} ${__dep_visibility}
        blake3_asm
    )
    if(BLAKE3_INSTALL)
        install(TARGETS ${__target}
            RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
            LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
            ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        )
        install(FILES BLAKE3/blake3.h DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
    endif()
endforeach()
