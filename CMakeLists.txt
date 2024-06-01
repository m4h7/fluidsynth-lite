# FluidSynth - A Software Synthesizer
#
# Copyright (C) 2003-2011 Peter Hanappe and others.
#
# This library is free software; you can redistribute it and/or
# modify it under the terms of the GNU Library General Public License
# as published by the Free Software Foundation; either version 2 of
# the License, or (at your option) any later version.
#
# This library is distributed in the hope that it will be useful, but
# WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the GNU
# Library General Public License for more details.
#
# You should have received a copy of the GNU Library General Public
# License along with this library; if not, write to the Free
# Foundation, Inc., 59 Temple Place, Suite 330, Boston, MA
# 02111-1307, USA

# CMake based build system. Pedro Lopez-Cabanillas <plcl@users.sf.net>

cmake_minimum_required(VERSION 3.18)

project(FluidSynth C)
set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake_admin)

# FluidSynth package name
set ( PACKAGE "fluidsynth" )

# FluidSynth package version
set ( FLUIDSYNTH_VERSION_MAJOR 1 )
set ( FLUIDSYNTH_VERSION_MINOR 1 )
set ( FLUIDSYNTH_VERSION_MICRO 6 )
set ( VERSION "${FLUIDSYNTH_VERSION_MAJOR}.${FLUIDSYNTH_VERSION_MINOR}.${FLUIDSYNTH_VERSION_MICRO}" )
set ( FLUIDSYNTH_VERSION "\"${VERSION}\"" )

# libfluidsynth - Library version
# *** NOTICE ***
# Update library version upon each release (follow these steps in order)
# if any source code changes: REVISION++
# if any interfaces added/removed/changed: REVISION=0
# if any interfaces removed/changed (compatibility broken): CURRENT++
# if any interfaces have been added: AGE++
# if any interfaces have been removed/changed (compatibility broken): AGE=0
# This is not exactly the same algorithm as the libtool one, but the results are the same.
set ( LIB_VERSION_CURRENT 1 )
set ( LIB_VERSION_AGE 5 )
set ( LIB_VERSION_REVISION 2 )
set ( LIB_VERSION_INFO
      "${LIB_VERSION_CURRENT}.${LIB_VERSION_AGE}.${LIB_VERSION_REVISION}" )

# Options disabled by default
option ( enable-floats "enable type float instead of double for DSP samples" off )
option ( enable-profiling "profile the dsp code" off )
option ( enable-trap-on-fpe "enable SIGFPE trap on Floating Point Exceptions" off )
option ( enable-fpe-check "enable Floating Point Exception checks and debug messages" off )
option ( enable-debug "enable debugging (default=no)" off )
option ( BUILD_SHARED_LIBS "Build a shared object or DLL" off )

# Initialize the library directory name suffix.
if (CMAKE_SIZEOF_VOID_P EQUAL 8)
  set (_init_lib_suffix "64")
else (CMAKE_SIZEOF_VOID_P EQUAL 8)
  set (_init_lib_suffix "")
endif (CMAKE_SIZEOF_VOID_P EQUAL 8)
set (LIB_SUFFIX ${_init_lib_suffix} CACHE STRING
      "library directory name suffix (32/64/nothing)")
mark_as_advanced(LIB_SUFFIX)

# Default install directory names
include (DefaultDirs)

# Basic C library checks
#include ( CheckSTDC )
include (CheckIncludeFile)
check_include_file(string.h HAVE_STRING_H)
check_include_file(stdlib.h HAVE_STDLIB_H)
check_include_file(stdio.h HAVE_STDIO_H)
check_include_file(stdint.h HAVE_STDINT_H)
check_include_file(stdbool.h HAVE_STDBOOL_H)
check_include_file(math.h HAVE_MATH_H)
check_include_file(errno.h HAVE_ERRNO_H)
check_include_file(stdarg.h HAVE_STDARG_H)
check_include_file(unistd.h HAVE_UNISTD_H)
check_include_file(memory.h HAVE_MEMORY_H)
check_include_file(sys/mman.h HAVE_SYS_MMAN_H)
check_include_file(sys/types.h HAVE_SYS_TYPES_H)
check_include_file(sys/time.h HAVE_SYS_TIME_H)
check_include_file(sys/stat.h HAVE_SYS_STAT_H)
check_include_file(sys/ioctl.h HAVE_SYS_IOCTL_H)
check_include_file(fcntl.h HAVE_FCNTL_H)
check_include_file(pthread.h HAVE_PTHREAD_H)

include(TestInline)
include(TestVLA)
include(TestBigEndian)
test_big_endian(WORDS_BIGENDIAN)

# Options for the GNU C compiler only
if (CMAKE_COMPILER_IS_GNUCC)
  if (NOT APPLE)
#    set (CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,--as-needed")
    set (CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -Wl,--no-undefined")
  endif()
endif()

# Windows
if (WIN32)
  set ( LIBFLUID_CPPFLAGS "-DFLUIDSYNTH_DLL_EXPORTS" )
  set ( FLUID_CPPFLAGS "-DFLUIDSYNTH_NOT_A_DLL" )
  set ( CMAKE_DEBUG_POSTFIX "_debug" )
  # MinGW compiler (a Windows GCC port)
  if ( MINGW )
    set ( MINGW32 1 )
    add_definitions ( -mms-bitfields )
    add_definitions ( -D_WIN32_WINNT=0x600 )
  endif  ( MINGW )
else ( WIN32 )
# Check PThreads, but not in Windows
  set ( HAVE_LIBPTHREAD on )
  set ( LIBFLUID_LIBS m pthread )
endif ( WIN32 )

# Apple Mac OSX
if ( ${CMAKE_SYSTEM} MATCHES "Darwin" )
  set ( DARWIN 1 )
  set ( CMAKE_INSTALL_NAME_DIR
        ${CMAKE_INSTALL_PREFIX}/${LIB_INSTALL_DIR}${LIB_SUFFIX} )
  if ( enable-coreaudio )
    check_include_file ( CoreAudio/AudioHardware.h COREAUDIO_FOUND )
    if ( COREAUDIO_FOUND )
      set ( COREAUDIO_SUPPORT ${COREAUDIO_FOUND} )
      set ( COREAUDIO_LIBS "-Wl,-framework,CoreAudio,-framework,AudioUnit" )
    endif ( COREAUDIO_FOUND )
  endif ( enable-coreaudio )
  if ( enable-coremidi )
    check_include_file ( CoreMIDI/MIDIServices.h COREMIDI_FOUND )
    if ( COREMIDI_FOUND )
      set ( COREMIDI_SUPPORT ${COREMIDI_FOUND} )
      set ( COREMIDI_LIBS "-Wl,-framework,CoreMIDI,-framework,CoreServices" )
    endif ( COREMIDI_FOUND )
  endif ( enable-coremidi )
  if ( enable-framework )
    set ( MACOSX_FRAMEWORK 1 )
  endif ( enable-framework )
endif ( ${CMAKE_SYSTEM} MATCHES "Darwin" )

if ( enable-floats )
    set ( WITH_FLOAT 1 )
endif ( enable-floats )

if ( enable-profiling )
    set ( WITH_PROFILING 1 )
endif ( enable-profiling )

if ( enable-ladspa )
  check_include_file ( ladspa.h LADSPA_SUPPORT )
  if ( LADSPA_SUPPORT )
    set ( LADSPA 1 )
    if ( CMAKE_DL_LIBS )
      set ( HAVE_LIBDL 1 )
      set ( LIBFLUID_LIBS "${LIBFLUID_LIBS};${CMAKE_DL_LIBS}" )
    endif ( CMAKE_DL_LIBS )
  endif ( LADSPA_SUPPORT )
endif ( enable-ladspa )

if ( enable-trap-on-fpe AND NOT APPLE AND NOT WIN32 )
    set ( ENABLE_TRAPONFPE 1 )
    set ( TRAP_ON_FPE 1 )
endif ( enable-trap-on-fpe AND NOT APPLE AND NOT WIN32 )

if ( enable-fpe-check )
    set ( ENABLE_FPECHECK 1 )
    set ( FPE_CHECK 1 )
endif ( enable-fpe-check )

if (CMAKE_BUILD_TYPE MATCHES "Debug")
  set (DEBUG 1)
endif()

# General configuration file
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/src/config.cmake
               ${CMAKE_CURRENT_BINARY_DIR}/config.h)
add_definitions(-DHAVE_CONFIG_H)

set(config_SOURCES ${CMAKE_CURRENT_BINARY_DIR}/config.h)

# Extra configuration file for MS VisualC compiler
if (MSVC)
  configure_file(${CMAKE_CURRENT_SOURCE_DIR}/src/config_win32.cmake
                 ${CMAKE_CURRENT_BINARY_DIR}/config_win32.h)
  set(config_SOURCES ${config_SOURCES} ${CMAKE_CURRENT_BINARY_DIR}/config_win32.h)
endif()

# Add public headers
set(public_HEADERS
    ${CMAKE_CURRENT_SOURCE_DIR}/include/fluidsynth/audio.h
    ${CMAKE_CURRENT_SOURCE_DIR}/include/fluidsynth/event.h
    ${CMAKE_CURRENT_SOURCE_DIR}/include/fluidsynth/gen.h
    ${CMAKE_CURRENT_SOURCE_DIR}/include/fluidsynth/log.h
    ${CMAKE_CURRENT_SOURCE_DIR}/include/fluidsynth/midi.h
    ${CMAKE_CURRENT_SOURCE_DIR}/include/fluidsynth/misc.h
    ${CMAKE_CURRENT_SOURCE_DIR}/include/fluidsynth/mod.h
    ${CMAKE_CURRENT_SOURCE_DIR}/include/fluidsynth/ramsfont.h
    ${CMAKE_CURRENT_SOURCE_DIR}/include/fluidsynth/seq.h
    ${CMAKE_CURRENT_SOURCE_DIR}/include/fluidsynth/seqbind.h
    ${CMAKE_CURRENT_SOURCE_DIR}/include/fluidsynth/settings.h
    ${CMAKE_CURRENT_SOURCE_DIR}/include/fluidsynth/sfont.h
    ${CMAKE_CURRENT_SOURCE_DIR}/include/fluidsynth/shell.h
    ${CMAKE_CURRENT_SOURCE_DIR}/include/fluidsynth/synth.h
    ${CMAKE_CURRENT_SOURCE_DIR}/include/fluidsynth/types.h
    ${CMAKE_CURRENT_SOURCE_DIR}/include/fluidsynth/voice.h
    ${CMAKE_CURRENT_BINARY_DIR}/include/fluidsynth/version.h
)

set (public_main_HEADER ${CMAKE_CURRENT_SOURCE_DIR}/include/fluidsynth.h)

# Process subdirectories
add_subdirectory(src)
add_subdirectory(include)

# CPack support
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "FluidSynth real-time synthesizer")
set(CPACK_PACKAGE_VENDOR "fluidsynth.org")
set(CPACK_PACKAGE_DESCRIPTION_FILE "${CMAKE_CURRENT_SOURCE_DIR}/README.md")
set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/COPYING")
set(CPACK_PACKAGE_VERSION_MAJOR ${FLUIDSYNTH_VERSION_MAJOR})
set(CPACK_PACKAGE_VERSION_MINOR ${FLUIDSYNTH_VERSION_MINOR})
set(CPACK_PACKAGE_VERSION_PATCH ${FLUIDSYNTH_VERSION_MICRO})

# source packages
set(CPACK_SOURCE_GENERATOR TGZ;TBZ2;ZIP)
set(CPACK_SOURCE_IGNORE_FILES "/.svn/;/build/;~$;.cproject;.project;/.settings/;${CPACK_SOURCE_IGNORE_FILES}")
set(CPACK_SOURCE_PACKAGE_FILE_NAME "${PACKAGE}-${VERSION}")
set(CPACK_SOURCE_STRIP_FILES OFF)

# binary packages
include(InstallRequiredSystemLibraries)
set(CPACK_GENERATOR STGZ;TGZ;TBZ2;ZIP)
set(CPACK_PACKAGE_NAME ${PACKAGE})
set(CPACK_STRIP_FILES ON)

include (CPack)

# set variables for including in other CMakeLists
set(FLUIDSYNTH_INCLUDE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/include" "${CMAKE_CURRENT_BINARY_DIR}/include" PARENT_SCOPE)
set(FLUIDSYNTH_LIBRARIES libfluidsynth PARENT_SCOPE)
