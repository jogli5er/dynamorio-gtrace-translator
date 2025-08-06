# **********************************************************
# Copyright (c) 2023-2025 Google, Inc.    All rights reserved.
# **********************************************************

# Redistribution and use in source and binary forms, with or without
# modification, are permitted provided that the following conditions are met:
#
# * Redistributions of source code must retain the above copyright notice,
#   this list of conditions and the following disclaimer.
#
# * Redistributions in binary form must reproduce the above copyright notice,
#   this list of conditions and the following disclaimer in the documentation
#   and/or other materials provided with the distribution.
#
# * Neither the name of Google, Inc. nor the names of its contributors may be
#   used to endorse or promote products derived from this software without
#   specific prior written permission.
#
# THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS"
# AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE
# IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE
# ARE DISCLAIMED. IN NO EVENT SHALL VMWARE, INC. OR CONTRIBUTORS BE LIABLE
# FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL
# DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR
# SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER
# CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT
# LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY
# OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH
# DAMAGE.

cmake_minimum_required(VERSION 3.14)

include(../../../../../make/policies.cmake NO_POLICY_SCOPE)

add_dr_defines()

get_filename_component(DRCACHESIM_SOURCE_DIR
                       ${CMAKE_CURRENT_SOURCE_DIR}/../../.. ABSOLUTE)

add_exported_library(drmemtrace_champsim_tracer STATIC
                     tracer.cpp
                     ${DRCACHESIM_SOURCE_DIR}/common/options.cpp)
set_property(TARGET drmemtrace_champsim_tracer PROPERTY POSITION_INDEPENDENT_CODE ON)
add_exported_library(tracer_creator SHARED tracer_creator.cpp)
target_link_libraries(tracer_creator drmemtrace_champsim_tracer)

place_shared_lib_in_lib_dir(tracer_creator)

add_executable(tracer_launcher tracer_launcher.cpp)
_DR_get_static_libc_list(static_libc)
target_link_libraries(tracer_launcher
                      drmemtrace_analyzer
                      drmemtrace_champsim_tracer
                      ${static_libc})
use_DynamoRIO_extension(tracer_launcher droption)
add_dependencies(tracer_launcher api_headers)

# These add_win32_flags() calls are necessary only for building the project inside
# the DR top level project.
add_win32_flags(drmemtrace_champsim_tracer OFF)
add_win32_flags(tracer_creator OFF)
add_win32_flags(tracer_launcher OFF)

function (write_config_file name dst libdir)
  if (X64)
    string(REPLACE "lib64" "lib32" alt_libdir ${libdir})
    set(CUR_BIT "64")
    set(ALT_BIT "32")
  else ()
    set(CUR_BIT "32")
    set(ALT_BIT "64")
    string(REPLACE "lib64" "lib32" alt_libdir ${libdir})
  endif ()
  file(GENERATE OUTPUT ${dst} CONTENT
"# analysis tool creator config file\n\
TOOL_NAME=${name}
CREATOR_BIN${CUR_BIT}=${libdir}/${LIB_PFX}${name}_creator${LIB_EXT}\n\
CREATOR_BIN${ALT_BIT}=${alt_libdir}/${LIB_PFX}${name}_creator${LIB_EXT}\n\
")
endfunction ()

set(TRACER_CONFIG_INSTALL ${PROJECT_BINARY_DIR}/champsim_tracer.drcachesim)
set(TRACER_CONFIG_BUILD ${PROJECT_BINARY_DIR}/tools/champsim_tracer.drcachesim)
write_config_file(champsim_tracer ${TRACER_CONFIG_BUILD} clients/${INSTALL_LIB})
write_config_file(champsim_tracer ${TRACER_CONFIG_INSTALL} tools/${INSTALL_LIB})
DR_install(FILES "${TRACER_CONFIG_INSTALL}" DESTINATION ${INSTALL_CLIENTS_BASE})


##################################################
# Tests

set(test_seconds 90)

if (X86 AND X64 AND ZIP_FOUND)
# Simple test to ensure champsim_tracer launcher keeps working.
  set(trace_dir
    "${PROJECT_SOURCE_DIR}/clients/drcachesim/tests/drmemtrace.threadsig.x64.tracedir")
  add_test(NAME tool.tracer_launcher
    COMMAND tracer_launcher -trace ${trace_dir})
  set_tests_properties(tool.tracer_launcher PROPERTIES TIMEOUT ${test_seconds})

  # Simple test to ensure champsim_tracer tool can be loaded by drcachesim.
  set(trace_dir
    "${PROJECT_SOURCE_DIR}/clients/drcachesim/tests/drmemtrace.threadsig.x64.tracedir")
  add_test(NAME drcachesim.empty_load
    COMMAND ${PROJECT_BINARY_DIR}/bin64/drrun -t drmemtrace -offline
     -tool champsim_tracer -indir ${trace_dir})
  set(empty_regex "Empty tool created\nEmpty tool results:")
  set_tests_properties(drcachesim.empty_load
    PROPERTIES PASS_REGULAR_EXPRESSION "${empty_regex}")

  # Simple test to ensure non-existent tool load by drcachesim should fail.
  add_test(NAME drcachesim.non-existent_load
    COMMAND ${PROJECT_BINARY_DIR}/bin64/drrun -t drmemtrace -offline
     -tool non-existent -indir ${trace_dir})
  set(nonexistent_regex "Usage error: unsupported analyzer type \"non-existent\". Please choose cache_simulator, miss_analyzer, TLB_simulator, histogram, reuse_distance, basic_counts, opcode_mix, syscall_mix, view, func_view, or some external analyzer.\nERROR: failed to initialize analyzer: Failed to create analysis tool:")
  set_tests_properties(drcachesim.non-existent_load
    PROPERTIES PASS_REGULAR_EXPRESSION "${nonexistent_regex}")
endif ()
