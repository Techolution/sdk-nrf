#
# Copyright (c) 2020 Nordic Semiconductor ASA
#
# SPDX-License-Identifier: LicenseRef-BSD-5-Clause-Nordic
#

cmake_minimum_required(VERSION 3.13)

set(EDGE_IMPULSE_DIR        ${CMAKE_BINARY_DIR}/edge_impulse)
set(EDGE_IMPULSE_SOURCE_DIR ${EDGE_IMPULSE_DIR}/src/edge_impulse_project)
set(EDGE_IMPULSE_BINARY_DIR ${EDGE_IMPULSE_DIR}/src/edge_impulse_project-build)
set(EDGE_IMPULSE_STAMP_DIR  ${EDGE_IMPULSE_DIR}/src/edge_impulse_project-stamp)
set(EDGE_IMPULSE_LIBRARY    ${EDGE_IMPULSE_BINARY_DIR}/libedge_impulse.a)

file(GENERATE OUTPUT ${EDGE_IMPULSE_DIR}/compile_options.$<COMPILE_LANGUAGE>.cmake CONTENT
"set(EI_$<COMPILE_LANGUAGE>_COMPILE_OPTIONS \"$<TARGET_PROPERTY:zephyr_interface,INTERFACE_COMPILE_OPTIONS>\")"
)

if(CONFIG_EDGE_IMPULSE_URI STREQUAL "")
  message(FATAL_ERROR "CONFIG_EDGE_IMPULSE_URI must be specified")
endif()

set(EI_URI ${CONFIG_EDGE_IMPULSE_URI})

if(NOT ${EI_URI} MATCHES "^[a-z]+://")
  if(NOT IS_ABSOLUTE ${EI_URI})
    message(FATAL_ERROR "CONFIG_EDGE_IMPULSE_URI cannot be a relative path")
  endif()
endif()

include(ExternalProject)
ExternalProject_Add(edge_impulse_project
    URL              ${EI_URI}
    HTTP_HEADER      "Accept: application/zip"
                     ${EI_API_KEY_HEADER}
    PREFIX           ${EDGE_IMPULSE_DIR}
    SOURCE_DIR       ${EDGE_IMPULSE_SOURCE_DIR}
    BINARY_DIR       ${EDGE_IMPULSE_BINARY_DIR}
    STAMP_DIR        ${EDGE_IMPULSE_STAMP_DIR}
    DOWNLOAD_NAME    edge_impulse_src.zip
    BUILD_BYPRODUCTS ${EDGE_IMPULSE_LIBRARY}
    PATCH_COMMAND    ${CMAKE_COMMAND} -E copy
                     ${CMAKE_CURRENT_LIST_DIR}/CMakeLists.ei.template
                     ${EDGE_IMPULSE_SOURCE_DIR}/CMakeLists.txt
    CMAKE_ARGS -DCMAKE_C_COMPILER=${CMAKE_C_COMPILER}
               -DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}
               -DCMAKE_AR=${CMAKE_AR}
               -DCMAKE_RANLIB=${CMAKE_RANLIB}
               -DEI_COMPILE_DEFINITIONS=$<TARGET_PROPERTY:zephyr_interface,INTERFACE_COMPILE_DEFINITIONS>
               -DEI_INCLUDE_DIRECTORIES=$<TARGET_PROPERTY:zephyr_interface,INTERFACE_INCLUDE_DIRECTORIES>
               -DEI_SYSTEM_INCLUDE_DIRECTORIES=$<TARGET_PROPERTY:zephyr_interface,INTERFACE_SYSTEM_INCLUDE_DIRECTORIES>
               -DEI_LIBRARY_NAME=edge_impulse
    INSTALL_COMMAND ""
    BUILD_ALWAYS True
    USES_TERMINAL_BUILD True
)

# This targets remove the `edge_impulse_project-download` stamp file, which
# causes the Edge impulse library to be fetched on each build invocation.
# Note: This also results in the `ALL` target to always be considered out-of-date.
if(CONFIG_EDGE_IMPULSE_DOWNLOAD_ALWAYS)
  add_custom_target(edge_impulse_project_download
                    COMMAND ${CMAKE_COMMAND} -E rm -f
                            ${EDGE_IMPULSE_STAMP_DIR}/edge_impulse_project-download
                    DEPENDS edge_impulse_project
  )
endif()

zephyr_interface_library_named(edge_impulse)
target_include_directories(edge_impulse INTERFACE
                           ${EDGE_IMPULSE_SOURCE_DIR}
                           ${EDGE_IMPULSE_SOURCE_DIR}/tflite-model
                           ${EDGE_IMPULSE_SOURCE_DIR}/model-parameters
                           ${EDGE_IMPULSE_SOURCE_DIR}/edge-impulse-sdk/
                           ${EDGE_IMPULSE_SOURCE_DIR}/edge-impulse-sdk/third_party/ruy
                           ${EDGE_IMPULSE_SOURCE_DIR}/edge-impulse-sdk/third_party/gemmlowp
                           ${EDGE_IMPULSE_SOURCE_DIR}/edge-impulse-sdk/third_party/flatbuffers/include
                           ${EDGE_IMPULSE_SOURCE_DIR}/edge-impulse-sdk/third_party
                           ${EDGE_IMPULSE_SOURCE_DIR}/edge-impulse-sdk/tensorflow
                           ${EDGE_IMPULSE_SOURCE_DIR}/edge-impulse-sdk/dsp
                           ${EDGE_IMPULSE_SOURCE_DIR}/edge-impulse-sdk/classifier
                           ${EDGE_IMPULSE_SOURCE_DIR}/edge-impulse-sdk/anomaly
                           ${EDGE_IMPULSE_SOURCE_DIR}/edge-impulse-sdk/CMSIS/NN/Include
                           ${EDGE_IMPULSE_SOURCE_DIR}/edge-impulse-sdk/CMSIS/DSP/PrivateInclude
                           ${EDGE_IMPULSE_SOURCE_DIR}/edge-impulse-sdk/CMSIS/DSP/Include
                           ${EDGE_IMPULSE_SOURCE_DIR}/edge-impulse-sdk/CMSIS/Core/Include
)

target_link_libraries(edge_impulse INTERFACE ${EDGE_IMPULSE_LIBRARY})

add_dependencies(edge_impulse
                 edge_impulse_project
                 edge_impulse_project_download
)

if(CONFIG_EI_NCS)
  zephyr_library_named(ei_ncs)
  zephyr_library_sources(ei_ncs.cpp)
  zephyr_library_link_libraries(edge_impulse)
endif()
