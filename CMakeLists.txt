# -*- mode: cmake; coding: utf-8 -*-
# This file is part of Pathie.
#
# Copyright © 2015, 2017, 2019 Marvin Gülker
#
# Redistribution and use in source and binary forms, with or without
# modification, are permitted provided that the following conditions are
# met:
#
# 1. Redistributions of source code must retain the above copyright
#    notice, this list of conditions and the following disclaimer.
#
# 2. Redistributions in binary form must reproduce the above copyright
#    notice, this list of conditions and the following disclaimer in the
#    documentation and/or other materials provided with the distribution.
#
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS
# “AS IS” AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT
# LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR
# A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT
# HOLDER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL,
# SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT
# LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE,
# DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY
# THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT
# (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE
# OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

cmake_minimum_required(VERSION 3.16)
set(CMAKE_CXX_STANDARD 17)

if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE "Release")
endif()

project(Pathie VERSION 0.1.3 LANGUAGES CXX)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

include(GNUInstallDirs)  # initializes ${CMAKE_INSTALL_*DIR} to sane defaults
include(CMakePackageConfigHelpers)

unset(PATHIE_VERSION_GIT)

########################################
# Flags & Options

option(PATHIE_BUILD_STREAM_REPLACEMENTS "Build the std::ifstream and std::ofstream replacements (experimental)." OFF)
option(PATHIE_ASSUME_UTF8_ON_UNIX "Assume file paths under UNIX to be always encoded in UTF-8." OFF)

########################################
# Extra flags

set(CMAKE_CXX_STANDARD 98)
set(CMAKE_CXX_EXTENSIONS OFF)

if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra ")
endif()

# We only support Vista upwards. See http://msdn.microsoft.com/en-us/library/windows/desktop/aa383745(v=vs.85).aspx
# for the macro values.
if (WIN32)
  add_definitions("-D_WIN32_WINNT=0x0600") # Vista
  add_definitions("-D_WIN32_IE=0x0800") # IE 8.0+
  add_definitions("-DWINVER=0x0600") # Vista
elseif(APPLE)
  add_definitions("-D_DARWIN_C_SOURCE")
  add_definitions("-D_PATHIE_UNIX=1")
elseif(UNIX)
  add_definitions("-D_PATHIE_UNIX=1")
  add_definitions("-D_POSIX_C_SOURCE=200112L")
endif()

add_definitions("-DPATHIE_VERSION_MAJOR=${CMAKE_PROJECT_VERSION_MAJOR}")
add_definitions("-DPATHIE_VERSION_MINOR=${CMAKE_PROJECT_VERSION_MINOR}")
add_definitions("-DPATHIE_VERSION_PATCH=${CMAKE_PROJECT_VERSION_PATCH}")

if (PATHIE_ASSUME_UTF8_ON_UNIX)
  add_definitions("-DPATHIE_ASSUME_UTF8_ON_UNIX=1")
endif()

########################################
# Adjustments to cmake’s defaults

set(CMAKE_BUILD_SHARED_LIBS On CACHE BOOL "Additionally to the static lib, build the shared lib")

########################################
# Source files

list(APPEND pathie_sources
  "src/errors.cpp"
  "src/entry_iterator.cpp"
  "src/path.cpp"
  "src/temp.cpp"
  "src/pathie.cpp")

if (PATHIE_BUILD_STREAM_REPLACEMENTS)
  list(APPEND pathie_sources
    "src/pathie_ifstream.cpp"
    "src/pathie_ofstream.cpp")
endif()

file(GLOB_RECURSE test_sources
  "test/*.cpp")

########################################
# Targets

# Libraries
add_library(pathie STATIC ${pathie_sources})
set_target_properties(pathie PROPERTIES
        VERSION "${PATHIE_VERSION_MAJOR}.${PATHIE_VERSION_MINOR}.${PATHIE_VERSION_PATCH}"
        SOVERSION "${PATHIE_VERSION_MAJOR}"
        OUTPUT_NAME pathie-cpp
        )
if(APPLE)
  target_link_libraries(pathie iconv)
endif()

if(WIN32)
  target_link_libraries(pathie shlwapi)

  if (CMAKE_BUILD_SHARED_LIBS)
    target_link_libraries(pathie-dynamic shlwapi)
  endif()
endif()

# Tests
# TODO: How to only do not test with crosscompilation?
#if (NOT(WIN32))
#  foreach(testfile ${test_sources})
#    get_filename_component(testtargetname "${testfile}" NAME_WE)
#    set(testtargetname "${testtargetname}.test")
#    message(STATUS "TESTFILE: ${testtargetname}")
#    #string(REPLACE ".cpp" ".test" testtargetname ${testfile})
#    add_executable(${testtargetname} ${testfile} ${Pathie_SOURCE_DIR}/test/testhelpers.hpp)
#  endforeach()
#endif()

########################################
# Installation information

target_include_directories(pathie
        PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
        PUBLIC $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
        )

install(TARGETS pathie
        EXPORT pathieTargets
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
        INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
        )

list(APPEND HDRS
        "include/pathie/errors.hpp"
        "include/pathie/entry_iterator.hpp"
        "include/pathie/path.hpp"
        "include/pathie/temp.hpp"
        "include/pathie/pathie.hpp")

if (PATHIE_BUILD_STREAM_REPLACEMENTS)
  list(APPEND HDRS
          "include/pathie/pathie_ifstream.hpp"
          "include/pathie/pathie_ofstream.hpp")
endif()

install(FILES ${HDRS} DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/pathie")

file(COPY ${HDRS} DESTINATION ${CMAKE_CURRENT_BINARY_DIR}/pathie)

install(EXPORT pathieTargets
        FILE pathieTargets.cmake
        NAMESPACE pathie::
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/pathie
        )

configure_package_config_file(${CMAKE_CURRENT_SOURCE_DIR}/Config.cmake.in
        "${CMAKE_CURRENT_BINARY_DIR}/pathieConfig.cmake"
        INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/pathie
        )

write_basic_package_version_file(
        "${CMAKE_CURRENT_BINARY_DIR}/pathieConfigVersion.cmake"
        VERSION "${CMAKE_PROJECT_VERSION}"
        COMPATIBILITY AnyNewerVersion
)

install(FILES
        "${CMAKE_CURRENT_BINARY_DIR}/pathieConfig.cmake"
        "${CMAKE_CURRENT_BINARY_DIR}/pathieConfigVersion.cmake"
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/pathie
        )

########################################
# Status message

message(STATUS "--------------- Build info summary ----------------")
message(STATUS "Pathie version:     ${CMAKE_PROJECT_VERSION_MAJOR}.${CMAKE_PROJECT_VERSION_MINOR}.${CMAKE_PROJECT_VERSION_PATCH}")
message(STATUS "Build type:         ${CMAKE_BUILD_TYPE}")
message(STATUS "Install prefix:     ${CMAKE_INSTALL_PREFIX}")
if (CMAKE_CROSSCOMPILING)
  message(STATUS "Crosscompiling:     Yes")
else()
  message(STATUS "Crosscompiling:     No")
endif()
if (UNIX)
  message(STATUS "Assume UTF-8:       ${PATHIE_ASSUME_UTF8_ON_UNIX}")
else()
  message(STATUS "Assume UTF-8:       option unavailable on this target")
endif()
message(STATUS "Stream replacments: ${PATHIE_BUILD_STREAM_REPLACEMENTS}")
message(STATUS "---------------------------------------------------")
