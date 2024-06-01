# FluidSynth - A Software Synthesizer
#
# Copyright (C) 2003-2010 Peter Hanappe and others.
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

# ************ library ************

set ( libfluidsynth_SOURCES
    utils/fluid_conv.c
    utils/fluid_conv.h
    utils/fluid_hash.c
    utils/fluid_hash.h
    utils/fluid_list.c
    utils/fluid_list.h
    utils/fluid_ringbuffer.c
    utils/fluid_ringbuffer.h
    utils/fluid_settings.c
    utils/fluid_settings.h
    utils/fluidsynth_priv.h
    utils/fluid_sys.c
    utils/fluid_sys.h
    sfloader/fluid_defsfont.c
    sfloader/fluid_defsfont.h
    sfloader/fluid_ramsfont.c
    sfloader/fluid_ramsfont.h
    sfloader/fluid_sfont.h
    rvoice/fluid_adsr_env.c
    rvoice/fluid_adsr_env.h
    rvoice/fluid_chorus.c
    rvoice/fluid_chorus.h
    rvoice/fluid_iir_filter.c
    rvoice/fluid_iir_filter.h
    rvoice/fluid_lfo.c
    rvoice/fluid_lfo.h
    rvoice/fluid_rvoice.h
    rvoice/fluid_rvoice.c
    rvoice/fluid_rvoice_dsp.c
    rvoice/fluid_rvoice_event.h
    rvoice/fluid_rvoice_event.c
    rvoice/fluid_rvoice_mixer.h
    rvoice/fluid_rvoice_mixer.c
    rvoice/fluid_phase.h
    rvoice/fluid_rev.c
    rvoice/fluid_rev.h
    synth/fluid_chan.c
    synth/fluid_chan.h
    synth/fluid_event.c
    synth/fluid_event_priv.h
    synth/fluid_event_queue.h
    synth/fluid_gen.c
    synth/fluid_gen.h
    synth/fluid_mod.c
    synth/fluid_mod.h
    synth/fluid_synth.c
    synth/fluid_synth.h
    synth/fluid_tuning.c
    synth/fluid_tuning.h
    synth/fluid_voice.c
    synth/fluid_voice.h
    midi/fluid_midi.c
    midi/fluid_midi.h
    midi/fluid_midi_router.c
    midi/fluid_midi_router.h
    midi/fluid_seqbind.c
    midi/fluid_seq.c
)

if (APPLE)
    list(APPEND libfluidsynth_SOURCES utils/apple/timing_mach.c)
endif()

# note: by default this target creates a shared object (or dll). To build a
# static library instead, set the option BUILD_SHARED_LIBS to FALSE.
add_library(libfluidsynth
    ${config_SOURCES}
    ${fluid_windows_SOURCES}
    ${libfluidsynth_SOURCES}
    ${public_HEADERS}
    ${public_main_HEADER}
)

if (MACOSX_FRAMEWORK)
  set_property(SOURCE ${public_HEADERS}
    PROPERTY MACOSX_PACKAGE_LOCATION Headers/fluidsynth
  )
  set_target_properties(libfluidsynth
    PROPERTIES
      OUTPUT_NAME "FluidSynth"
      FRAMEWORK TRUE
      PUBLIC_HEADER "${public_main_HEADER}"
      FRAMEWORK_VERSION "${LIB_VERSION_CURRENT}"
      INSTALL_NAME_DIR ""
      VERSION ${LIB_VERSION_INFO}
      SOVERSION ${LIB_VERSION_CURRENT}
  )
else(MACOSX_FRAMEWORK)
  set_target_properties ( libfluidsynth
    PROPERTIES
      PREFIX "lib"
      OUTPUT_NAME "fluidsynth"
      VERSION ${LIB_VERSION_INFO}
      SOVERSION ${LIB_VERSION_CURRENT}
  )
endif(MACOSX_FRAMEWORK)

if (LIBFLUID_CPPFLAGS)
  set_target_properties(libfluidsynth
    PROPERTIES COMPILE_FLAGS ${LIBFLUID_CPPFLAGS} )
endif (LIBFLUID_CPPFLAGS)

target_link_libraries(libfluidsynth ${LIBFLUID_LIBS})

if (WIN32)
  target_link_libraries(libfluidsynth dsound winmm ws2_32)
endif()

if ("${CMAKE_C_COMPILER_ID}" MATCHES "Clang")
  target_compile_options(libfluidsynth PRIVATE
    -Wall
    -Weverything
    -Wno-unsafe-buffer-usage
    -Wno-padded)
elseif (CMAKE_COMPILER_IS_GNUCC)
  target_compile_options(libfluidsynth PRIVATE -Wall -Wextra -Wconversion)
  if (CMAKE_C_COMPILER_VERSION VERSION_GREATER 14.0)
    target_compile_options(libfluidsynth PRIVATE -fanalyzer)
  endif()
elseif ("${CMAKE_C_COMPILER_ID}" MATCHES "MSVC")
  target_compile_options(libfluidsynth PRIVATE /Wall)
endif()

#target_compile_features(libfluidsynth PRIVATE c_std_99)
set_target_properties(libfluidsynth PROPERTIES C_EXTENSIONS OFF)
set_target_properties(libfluidsynth PROPERTIES C_VISIBILITY_PRESET hidden)

target_compile_definitions(libfluidsynth PRIVATE
    $<$<CONFIG:Debug>:
        DEBUG=1
    >
)

target_compile_definitions(libfluidsynth PRIVATE
    $<$<CONFIG:Release>:
        NDEBUG=1
    >
)

set(CFLAGS_DEBUG -g)
set(CFLAGS_DEBUG_GCC -ggdb3)

target_compile_options(libfluidsynth PRIVATE
    $<$<CONFIG:Debug>:${CFLAGS_DEBUG}>)

target_compile_options(libfluidsynth PRIVATE
    $<$<AND:$<CONFIG:Debug>,$<C_COMPILER_ID:GNU>>:${CFLAGS_DEBUG}>)

set(CFLAGS_RELEASE -O3 -finline-functions)
set(CFLAGS_RELEASE_GCC -funroll-all-loops)

target_compile_options(libfluidsynth PRIVATE
    $<$<AND:$<CONFIG:Release>,$<C_COMPILER_ID:GNU,AppleClang,Clang>>:${CFLAGS_RELEASE}>)

target_compile_options(libfluidsynth PRIVATE
    $<$<AND:$<CONFIG:Release>,$<C_COMPILER_ID:GNU>>:${CFLAGS_RELEASE_GCC}>)

target_include_directories(
  libfluidsynth PRIVATE
    ${CMAKE_CURRENT_BINARY_DIR}/.. # config.h
#    ${CMAKE_CURRENT_SOURCE_DIR}
    ${CMAKE_CURRENT_SOURCE_DIR}/synth
    ${CMAKE_CURRENT_SOURCE_DIR}/rvoice
    ${CMAKE_CURRENT_SOURCE_DIR}/midi
    ${CMAKE_CURRENT_SOURCE_DIR}/utils
    ${CMAKE_CURRENT_SOURCE_DIR}/sfloader
    ${CMAKE_CURRENT_SOURCE_DIR}/../include
    ${CMAKE_CURRENT_BINARY_DIR}/../include # version.h
)

install(TARGETS libfluidsynth
  LIBRARY DESTINATION lib
  ARCHIVE DESTINATION lib)
