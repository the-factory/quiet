cmake_minimum_required (VERSION 2.6)
project (Quiet)
include(CheckLibraryExists)

set(QUIET_PROFILES_LOCATION "${CMAKE_INSTALL_PREFIX}/share/quiet/")
add_definitions(-DQUIET_PROFILES_LOCATION="${QUIET_PROFILES_LOCATION}/quiet-profiles.json")

if (EMSCRIPTEN)
  set(CMAKE_LIBRARY_PATH ${CMAKE_LIBRARY_PATH} "$ENV{EMSCRIPTEN}/system/local/lib/")
endif()

if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    add_definitions(-DQUIET_DEBUG)
    set(CMAKE_C_FLAGS ${CMAKE_C_FLAGS} "-g -O0")
    set(CMAKE_EXE_LINKER_FLAGS ${CMAKE_EXE_LINKER_FLAGS} "-Wl,-no_pie")
else()
    if("$CMAKE_C_COMPILER_ID}" STREQUAL "Clang")
      set(CMAKE_C_FLAGS ${CMAKE_C_FLAGS} "-Ofast -flto")
      set(CMAKE_EXE_LINKER_FLAGS ${CMAKE_EXE_LINKER_FLAGS} "-Ofast -flto")
    else()
      set(CMAKE_C_FLAGS ${CMAKE_C_FLAGS} "-O2")
    endif()
endif()

include_directories(${CMAKE_SOURCE_DIR}/include)

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -std=c99 -Wpedantic -Werror -Wall -D_XOPEN_SOURCE=700")
set(SRCFILES src/demodulator.c src/modulator.c src/utility.c src/decoder.c src/encoder.c src/profile.c src/ring.c)
set(LIBRARY_OUTPUT_PATH ${PROJECT_BINARY_DIR}/lib)
set(EXECUTABLE_OUTPUT_PATH ${PROJECT_BINARY_DIR}/bin)

find_library(LIQUID liquid REQUIRED)
if (LIQUID)
    check_library_exists(LIQUID flexframesync_set_header_len,flexframegen_set_header_len,ofdmflexframesync_set_header_len,ofdmflexframegen_set_header_len,resamp_rrrf_execute_output_block,liquid_pack_array_block "" LIQUID_DEVEL_FUNCTIONS)
    if(NOT LIQUID_DEVEL_FUNCTIONS)
        unset(LIQUID_DEVEL_FUNCTIONS CACHE)
        message(FATAL_ERROR "
libquiet requires the devel branch of libliquid but
  found a different branch installed
fetch the devel branch using
    git clone https://github.com/quiet/liquid-dsp.git -b devel --single-branch
and install it before continuing")
    endif()
else()
    unset(LIQUID CACHE)
    message(FATAL_ERROR "
libquiet requires libliquid but cannot find it
fetch the devel branch using
    git clone https://github.com/quiet/liquid-dsp.git -b devel --single-branch
and install it before continuing")
endif()

find_library(JANSSON jansson REQUIRED)
if (NOT JANSSON)
    unset(JANSSON CACHE)
    message(FATAL_ERROR "
libquiet requires libjansson but cannot find it
fetch the latest jansson release from
    http://www.digip.org/jansson/
and install it before continuing")
endif()

add_library(quiet_static ${SRCFILES})
target_link_libraries(quiet_static LIQUID JANSSON)
add_library(quiet MODULE ${SRCFILES})
target_link_libraries(quiet LIQUID JANSSON)

add_custom_target(lib DEPENDS quiet quiet_static)

if(EMSCRIPTEN)
  add_custom_target(quiet-profiles ALL COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_SOURCE_DIR}/quiet-profiles.json ${CMAKE_BINARY_DIR}/quiet-profiles.json)
  add_custom_target(quiet-js ALL COMMAND emcc -v -O2 $<TARGET_FILE:quiet> -o quiet-emscripten.js -s MAIN_MODULE=2 -s EXPORTED_FUNCTIONS=\"['_quiet_decoder_consume', '_quiet_decoder_create', '_quiet_decoder_flush', '_quiet_decoder_recv', '_quiet_decoder_destroy', '_quiet_encoder_emit', '_quiet_encoder_create', '_quiet_encoder_destroy', '_quiet_encoder_send', '_quiet_encoder_get_frame_len', '_quiet_encoder_profile_str', '_quiet_decoder_profile_str', '_quiet_encoder_opt_set_sample_rate', '_quiet_decoder_opt_set_sample_rate', '_quiet_encoder_clamp_frame_len', '_quiet_decoder_checksum_fails']\" -lliquid -ljansson)
else() # EMSCRIPTEN
add_custom_target(quiet-h ALL COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_SOURCE_DIR}/include/quiet.h ${CMAKE_BINARY_DIR}/include/quiet.h)

add_custom_target(quiet-profiles ALL COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_SOURCE_DIR}/quiet-profiles.json ${CMAKE_BINARY_DIR}/share/quiet-profiles.json)

set(buildable_programs "")

find_library(SNDFILE sndfile)
if (SNDFILE)
    add_executable(quiet_encode_file programs/encode_file.c)
    target_link_libraries(quiet_encode_file quiet_static SNDFILE)
    set(buildable_programs ${buildable_programs} quiet_encode_file)

    add_executable(quiet_decode_file programs/decode_file.c)
    target_link_libraries(quiet_decode_file quiet_static SNDFILE)
    set(buildable_programs ${buildable_programs} quiet_decode_file)
else()
    unset(SNDFILE CACHE)
    message(WARNING "

libquiet could not find libsndfile. this will prevent libquiet
from building its file generating programs. you can get libsndfile from
    http://www.mega-nerd.com/libsndfile/#Download

")
endif()

find_library(PORTAUDIO portaudio)
if (PORTAUDIO)
    add_executable(quiet_encode_soundcard programs/encode_soundcard.c)
    target_link_libraries(quiet_encode_soundcard quiet_static PORTAUDIO)
    set(buildable_programs ${buildable_programs} quiet_encode_soundcard)

    add_executable(quiet_decode_soundcard programs/decode_soundcard.c)
    target_link_libraries(quiet_decode_soundcard quiet_static PORTAUDIO)
    set(buildable_programs ${buildable_programs} quiet_decode_soundcard)
else()
    unset(PORTAUDIO CACHE)
    message(WARNING "

libquiet could not find portaudio. this will prevent libquiet
from building its sound streaming programs. you can get portaudio from
    http://www.portaudio.com/download.html

")
endif()

add_custom_target(programs DEPENDS ${buildable_programs})

install(FILES ${CMAKE_BINARY_DIR}/share/quiet-profiles.json DESTINATION ${QUIET_PROFILES_LOCATION})
install(TARGETS quiet_encode_soundcard quiet_encode_file
                quiet_decode_soundcard quiet_decode_file
                quiet quiet_static
        RUNTIME DESTINATION bin
        LIBRARY DESTINATION lib
        ARCHIVE DESTINATION lib)

add_custom_target(docs COMMAND cldoc generate -- --output docs/html --language=c ${CMAKE_BINARY_DIR}/include/quiet.h DEPENDS quiet-h)

add_executable(integration_test_runner EXCLUDE_FROM_ALL tests/integration.c)
target_link_libraries(integration_test_runner quiet_static)
set_target_properties(integration_test_runner PROPERTIES RUNTIME_OUTPUT_DIRECTORY "tests")
add_test(NAME integration_test WORKING_DIRECTORY "${CMAKE_BINARY_DIR}/tests" COMMAND integration_test_runner)

add_custom_target(test-profiles COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_SOURCE_DIR}/tests/test-profiles.json ${CMAKE_BINARY_DIR}/tests/test-profiles.json)
add_custom_target(cp-test-profiles DEPENDS test-profiles)
add_dependencies(integration_test_runner cp-test-profiles)

add_custom_target(test_runners DEPENDS integration_test_runner)
add_custom_target(check COMMAND ${CMAKE_CTEST_COMMAND} DEPENDS test_runners)
enable_testing()
endif() # EMSCRIPTEN
