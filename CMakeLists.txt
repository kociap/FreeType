# CMakeLists.txt
#
# Copyright (C) 2013-2019 by
# David Turner, Robert Wilhelm, and Werner Lemberg.
#
# Written originally by John Cary <cary@txcorp.com>
#
# This file is part of the FreeType project, and may only be used, modified,
# and distributed under the terms of the FreeType project license,
# LICENSE.TXT.  By continuing to use, modify, or distribute this file you
# indicate that you have read the license and understand and accept it
# fully.
#
#
# The following will 1. create a build directory and 2. change into it and
# call cmake to configure the build with default parameters as a static
# library.
#
#   cmake -E make_directory build
#   cmake -E chdir build cmake ..
#
# For a dynamic library, use
#
#   cmake -E chdir build cmake -D BUILD_SHARED_LIBS:BOOL=true ..
#
# For a framework on OS X, use
#
#   cmake -E chdir build cmake -G Xcode -D BUILD_FRAMEWORK:BOOL=true ..
#
# For an iOS static library, use
#
#   cmake -E chdir build cmake -G Xcode -D IOS_PLATFORM=OS ..
#
# or
#
#   cmake -E chdir build cmake -G Xcode -D IOS_PLATFORM=SIMULATOR ..
#
# or
#
#   cmake -E chdir build cmake -G Xcode -D IOS_PLATFORM=SIMULATOR64 ..
#
# Finally, build the project with:
#
#   cmake --build build
#
# Install it with
#
#   (sudo) cmake --build build --target install
#
# A binary distribution can be made with
#
#   cmake --build build --config Release --target package
#
# Please refer to the cmake manual for further options, in particular, how
# to modify compilation and linking parameters.
#
# Some notes.
#
# . `cmake' creates configuration files in
#
#     <build-directory>/include/freetype/config
#
#   which should be further modified if necessary.
#
# . You can use `cmake' directly on a freshly cloned FreeType git
#   repository.
#
# . `CMakeLists.txt' is provided as-is since it is normally not used by the
#   developer team.
#
# . Set the `FT_WITH_ZLIB', `FT_WITH_BZIP2', `FT_WITH_PNG', and
#   `FT_WITH_HARFBUZZ' CMake variables to `ON' to force using a dependency.
#   Leave a variable undefined (which is the default) to use the dependency
#   only if it is available.  Set `CMAKE_DISABLE_FIND_PACKAGE_HarfBuzz=TRUE' to
#   disable a dependency completely (CMake package name, so `BZip2' instead of
#   `BZIP2'). Example:
#
#     cmake -DFT_WITH_ZLIB=ON -DCMAKE_DISABLE_FIND_PACKAGE_HarfBuzz=TRUE [...]
#
# . Installation of FreeType can be controlled with the CMake variables
#   `SKIP_INSTALL_HEADERS', `SKIP_INSTALL_LIBRARIES', and `SKIP_INSTALL_ALL'
#   (this is compatible with the same CMake variables in zlib's CMake
#   support).

# FreeType explicitly marks the API to be exported and relies on the compiler
# to hide all other symbols. CMake supports a C_VISBILITY_PRESET property
# starting with 2.8.12.
cmake_minimum_required(VERSION 3.15)

# Allow symbol visibility settings also on static libraries. CMake < 3.3
# only sets the propery on a shared library build.
cmake_policy(SET CMP0063 NEW)

include(CheckIncludeFile)

# CMAKE_TOOLCHAIN_FILE must be set before `project' is called, which
# configures the base build environment and references the toolchain file
if (APPLE)
  message(FATAL_ERROR "platform not supported")
else ()
  if (DEFINED IOS_PLATFORM)
    message(FATAL_ERROR "IOS_PLATFORM is not supported on this platform")
  endif ()
endif ()


project(freetype C)

set(VERSION_MAJOR "2")
set(VERSION_MINOR "10")
set(VERSION_PATCH "0")

# SOVERSION scheme: CURRENT.AGE.REVISION
#   If there was an incompatible interface change:
#     Increment CURRENT. Set AGE and REVISION to 0
#   If there was a compatible interface change:
#     Increment AGE. Set REVISION to 0
#   If the source code was changed, but there were no interface changes:
#     Increment REVISION.
set(LIBRARY_VERSION "6.16.0")
set(LIBRARY_SOVERSION "6")

# These options mean "require x and complain if not found". They'll get
# optionally found anyway. Use `-DCMAKE_DISABLE_FIND_PACKAGE_x=TRUE` to disable
# searching for a packge entirely (x is the CMake package name, so "BZip2"
# instead of "BZIP2").
option(FT_WITH_ZLIB "Use system zlib instead of internal library." OFF)
option(FT_WITH_BZIP2 "Support bzip2 compressed fonts." OFF)
option(FT_WITH_PNG "Support PNG compressed OpenType embedded bitmaps." OFF)
option(FT_WITH_HARFBUZZ "Improve auto-hinting of OpenType fonts." OFF)


# Disallow in-source builds
if ("${CMAKE_BINARY_DIR}" STREQUAL "${CMAKE_SOURCE_DIR}")
  message(FATAL_ERROR
    "In-source builds are not permitted!  Make a separate folder for"
    " building, e.g.,\n"
    "  cmake -E make_directory build\n"
    "  cmake -E chdir build cmake ..\n"
    "Before that, remove the files created by this failed run with\n"
    "  cmake -E remove CMakeCache.txt\n"
    "  cmake -E remove_directory CMakeFiles")
endif ()


# Add local cmake modules
list(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/builds/cmake)


# Find dependencies
if (FT_WITH_HARFBUZZ)
  find_package(HarfBuzz 1.3.0 REQUIRED)
else ()
  find_package(HarfBuzz 1.3.0)
endif ()

if (FT_WITH_PNG)
  find_package(PNG REQUIRED)
else ()
  find_package(PNG)
endif ()

if (FT_WITH_ZLIB)
  find_package(ZLIB REQUIRED)
else ()
  find_package(ZLIB)
endif ()

if (FT_WITH_BZIP2)
  find_package(BZip2 REQUIRED)
else ()
  find_package(BZip2)
endif ()

file(GLOB PUBLIC_HEADERS "include/ft2build.h" "include/freetype/*.h")
file(GLOB PUBLIC_CONFIG_HEADERS "include/freetype/config/*.h")
file(GLOB PRIVATE_HEADERS "include/freetype/internal/*.h")


set(BASE_SRCS
    src/autofit/autofit.c
    src/base/ftbase.c
    src/base/ftbbox.c
    src/base/ftbdf.c
    src/base/ftbitmap.c
    src/base/ftcid.c
    src/base/ftfstype.c
    src/base/ftgasp.c
    src/base/ftglyph.c
    src/base/ftgxval.c
    src/base/ftinit.c
    src/base/ftmm.c
    src/base/ftotval.c
    src/base/ftpatent.c
    src/base/ftpfr.c
    src/base/ftstroke.c
    src/base/ftsynth.c
    src/base/ftsystem.c
    src/base/fttype1.c
    src/base/ftwinfnt.c
    src/bdf/bdf.c
    src/bzip2/ftbzip2.c
    src/cache/ftcache.c
    src/cff/cff.c
    src/cid/type1cid.c
    src/gzip/ftgzip.c
    src/lzw/ftlzw.c
    src/pcf/pcf.c
    src/pfr/pfr.c
    src/psaux/psaux.c
    src/pshinter/pshinter.c
    src/psnames/psnames.c
    src/raster/raster.c
    src/sfnt/sfnt.c
    src/smooth/smooth.c
    src/truetype/truetype.c
    src/type1/type1.c
    src/type42/type42.c
    src/winfonts/winfnt.c
)

if (WIN32)
  enable_language(RC)
  list(APPEND BASE_SRCS builds/windows/ftdebug.c
                        src/base/ftver.rc)
elseif (WINCE)
  list(APPEND BASE_SRCS builds/wince/ftdebug.c)
else ()
  list(APPEND BASE_SRCS src/base/ftdebug.c)
endif ()


if (NOT DISABLE_FORCE_DEBUG_POSTFIX)
  set(CMAKE_DEBUG_POSTFIX d)
endif()


add_library(freetype
  ${PUBLIC_HEADERS}
  ${PUBLIC_CONFIG_HEADERS}
  ${PRIVATE_HEADERS}
  ${BASE_SRCS}
)

if(ZLIB_FOUND)
    target_compile_definitions(freetype PUBLIC FT_CONFIG_OPTION_SYSTEM_ZLIB=1)
    target_link_libraries(freetype PRIVATE ${ZLIB_LIBRARIES})
    target_include_directories(freetype PRIVATE ${ZLIB_INCLUDE_DIRS})
endif()

if(BZIP2_FOUND)
    target_compile_definitions(freetype PUBLIC FT_CONFIG_OPTION_USE_BZIP2=1)
    target_link_libraries(freetype PRIVATE ${BZIP2_LIBRARIES})
    target_include_directories(freetype PRIVATE ${BZIP2_INCLUDE_DIR}) # not BZIP2_INCLUDE_DIRS
endif()

if(PNG_FOUND)
    target_compile_definitions(freetype PUBLIC FT_CONFIG_OPTION_USE_PNG=1)
endif()

if(HARFBUZZ_FOUND)
    target_compile_definitions(freetype PUBLIC FT_CONFIG_OPTION_USE_HARFBUZZ=1)
endif()

# Create the configuration file
if(UNIX)
    check_include_file("unistd.h" HAVE_UNISTD_H)
    check_include_file("fcntl.h" HAVE_FCNTL_H)
    check_include_file("stdint.h" HAVE_STDINT_H)

    if(HAVE_UNISTD_H)
        target_compile_definitions(freetype PUBLIC HAVE_UNISTD_H=1)
    endif()

    if(HAVE_FCNTL_H)
        target_compile_definitions(freetype PUBLIC HAVE_FCNTL_H=1)
    endif()

    if(HAVE_STDINT_H)
        target_compile_definitions(freetype PUBLIC HAVE_STDINT_H=1)
    endif()
endif()


set_target_properties(
  freetype PROPERTIES
    C_VISIBILITY_PRESET hidden)

target_compile_definitions(
  freetype PRIVATE FT2_BUILD_LIBRARY)

if (WIN32)
  target_compile_definitions(
    freetype PRIVATE _CRT_SECURE_NO_WARNINGS _CRT_NONSTDC_NO_WARNINGS)
  if (BUILD_SHARED_LIBS)
    target_compile_definitions(
      freetype PRIVATE DLL_EXPORT)
  endif ()
endif ()

if (BUILD_SHARED_LIBS)
  set_target_properties(freetype PROPERTIES
    VERSION ${LIBRARY_VERSION}
    SOVERSION ${LIBRARY_SOVERSION})
endif ()

# Pick up ftconfig.h and ftoption.h generated above, first.
target_include_directories(freetype
    PUBLIC
    "${CMAKE_CURRENT_SOURCE_DIR}/include"
    PRIVATE
    "${CMAKE_CURRENT_SOURCE_DIR}/include"
)
