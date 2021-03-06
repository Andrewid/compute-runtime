# Copyright (c) 2018, Intel Corporation
#
# Permission is hereby granted, free of charge, to any person obtaining a
# copy of this software and associated documentation files (the "Software"),
# to deal in the Software without restriction, including without limitation
# the rights to use, copy, modify, merge, publish, distribute, sublicense,
# and/or sell copies of the Software, and to permit persons to whom the
# Software is furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included
# in all copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS
# OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL
# THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR
# OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE,
# ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR
# OTHER DEALINGS IN THE SOFTWARE.

project(cloc)

set(CLOC_SRCS_LIB
  ${IGDRCL_SOURCE_DIR}/offline_compiler/offline_compiler.cpp
  ${IGDRCL_SOURCE_DIR}/offline_compiler/offline_compiler.h
  ${IGDRCL_SOURCE_DIR}/offline_compiler/options.cpp
  ${IGDRCL_SOURCE_DIR}/offline_compiler/helper.cpp
  ${IGDRCL_SOURCE_DIR}/runtime/compiler_interface/create_main.cpp
  ${IGDRCL_SOURCE_DIR}/runtime/helpers/hw_info.cpp
  ${IGDRCL_SOURCE_DIR}/runtime/platform/extensions.h
  ${IGDRCL_SOURCE_DIR}/runtime/platform/extensions.cpp
  ${IGDRCL_SOURCE_DIR}/runtime/helpers/file_io.cpp
  ${IGDRCL_SOURCE_DIR}/runtime/helpers/abort.cpp
  ${IGDRCL_SOURCE_DIR}/runtime/helpers/debug_helpers.cpp
)

if(WIN32)
  list(APPEND CLOC_SRCS_LIB
    ${IGDRCL_SOURCE_DIR}/runtime/os_interface/windows/os_library.cpp
    ${IGDRCL_SOURCE_DIR}/runtime/dll/windows/options.cpp
  )
else()
  list(APPEND CLOC_SRCS_LIB
    ${IGDRCL_SOURCE_DIR}/runtime/os_interface/linux/os_library.cpp
    ${IGDRCL_SOURCE_DIR}/runtime/dll/linux/options.cpp
  )
endif()

set(CLOC_LIB_FLAGS_DEFINITIONS
  -DCIF_HEADERS_ONLY_BUILD
)

set(OPTIONAL_RUNTIME_GENX_FILES
  hw_info.cpp
)

macro(macro_for_each_platform)
  list(APPEND CLOC_SRCS_LIB
    ${IGDRCL_SOURCE_DIR}/runtime/${GEN_TYPE_LOWER}/hw_info_${PLATFORM_IT_LOWER}.cpp
    ${IGDRCL_SOURCE_DIR}/runtime/${GEN_TYPE_LOWER}/enable_${PLATFORM_IT_LOWER}.cpp
  )
endmacro()

macro(macro_for_each_gen)
  foreach(SRC_IT ${OPTIONAL_RUNTIME_GENX_FILES})
    set(SRC_FILE ${IGDRCL_SOURCE_DIR}/runtime/${GEN_TYPE_LOWER}/${SRC_IT})
    if(EXISTS ${SRC_FILE})
      list(APPEND CLOC_SRCS_LIB ${SRC_FILE})
    endif()
  endforeach()
  apply_macro_for_each_platform()
endmacro()

apply_macro_for_each_gen("SUPPORTED")

set(CLOC_SRCS
  ${CLOC_SRCS_LIB}
  ${CMAKE_CURRENT_SOURCE_DIR}/main.cpp
  ${CMAKE_CURRENT_SOURCE_DIR}/CMakeLists.txt
)
add_executable(cloc ${CLOC_SRCS})
create_project_source_tree(cloc ${IGDRCL_SOURCE_DIR}/runtime)

set(CLOC_INCLUDES
  ${IGC_OCL_ADAPTOR_DIR}
  ${CIF_BASE_DIR}
  ${HW_SRC_INCLUDE_PATH}
  ${UMKM_SHAREDDATA_INCLUDE_PATHS}
  ${KHRONOS_HEADERS_DIR}
  ${IGDRCL__IGC_INCLUDE_DIR}
  ${THIRD_PARTY_DIR}
)

target_include_directories(cloc BEFORE PRIVATE ${CLOC_INCLUDES})

target_compile_definitions(cloc PUBLIC ${CLOC_LIB_FLAGS_DEFINITIONS} ${SUPPORTED_GEN_FLAGS_DEFINITONS} DEFAULT_PLATFORM=${DEFAULT_SUPPORTED_PLATFORM})

if(UNIX)
  target_link_libraries(cloc dl pthread)
endif()

target_link_libraries(cloc elflib)

set_target_properties(cloc PROPERTIES FOLDER "offline_compiler")
set_property(TARGET cloc APPEND_STRING PROPERTY COMPILE_FLAGS ${ASAN_FLAGS} ${TSAN_FLAGS})

add_custom_target(copy_compiler_files DEPENDS ${IGDRCL__IGC_TARGETS})
set_target_properties(copy_compiler_files PROPERTIES FOLDER "opencl runtime")

foreach(TARGET_tmp ${IGDRCL__IGC_TARGETS})
  add_custom_command(
    TARGET copy_compiler_files
    PRE_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_if_different $<TARGET_FILE:${TARGET_tmp}> $<TARGET_FILE_DIR:cloc>
)
endforeach()

set(CLOC_SRCS_LIB ${CLOC_SRCS_LIB} PARENT_SCOPE)
