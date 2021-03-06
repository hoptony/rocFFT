# #############################################################################
# Copyright (c) 2016 - present Advanced Micro Devices, Inc. All rights reserved.
#
# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included in
# all copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.  IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
# THE SOFTWARE.
# #############################################################################


# A helper function to prefix a source list of files with a common
# path into a new list (non-destructive)
function( prepend_path prefix source_list_of_files return_list_of_files )
  foreach( file ${${source_list_of_files}} )
    if(IS_ABSOLUTE ${file} )
      list( APPEND new_list ${file} )
    else( )
      list( APPEND new_list ${prefix}/${file} )
    endif( )
  endforeach( )
  set( ${return_list_of_files} ${new_list} PARENT_SCOPE )
endfunction( )


# This builds the generator executable
add_subdirectory( generator )

# This includes an explicit list (all possible) of files the generator
# currently outputs
include( generator/generated-kernels.cmake )

set( generator_pattern all CACHE STRING "FFT kernels to generate" )
set_property( CACHE generator_pattern PROPERTY STRINGS pow2 pow3 pow5
  "pow2,3" "pow2,5" "pow3,5" all )

set( gen_headers ${kernels_launch} ${kernels_all} )
if( generator_pattern STREQUAL "pow2" )
    set( gen_headers ${kernels_launch} ${kernels_pow2} )
elseif( generator_pattern STREQUAL "pow3" )
    set( gen_headers ${kernels_launch} ${kernels_pow3} )
elseif( generator_pattern STREQUAL "pow5" )
    set( gen_headers ${kernels_launch} ${kernels_pow5} )
endif()

# add_custom_command will create source output files, that are used in
# a target below
# In order for the dependencies to work in cmake, add_custom_command
# must be in the same CMakeLists.txt file as the target
# [rocfft-device]
add_custom_command(
  OUTPUT ${gen_headers}
  COMMAND rocfft-kernel-generator ${generator_pattern}
  ${small_kernels_group_num}
  DEPENDS rocfft-kernel-generator
  COMMENT "Generator producing device kernels for rocfft-device"
)

# The following is a list of implementation files defining the library
set( rocfft_device_source
  transpose.cpp
  bluestein.cpp
  real2complex_embed.cpp
  complex2real_embed.cpp
  realcomplex_even.cpp
  realcomplex_pair.cpp
)

prepend_path( "../.."
  rocfft_headers_public relative_rocfft_device_headers_public )

if(NOT SINGLELIB)
  add_library( rocfft-device
    ${rocfft_device_source}
    ${relative_rocfft_device_headers_public}
    ${gen_headers} )
else()
  # Compile the device lib as a static library, which is then linked
  # into librocfft.so Useful for testing purposes.
  add_library( rocfft-device STATIC
    ${rocfft_device_source}
    ${relative_rocfft_device_headers_public}
    ${gen_headers}
    )
  set_property(TARGET rocfft-device PROPERTY POSITION_INDEPENDENT_CODE ON)
endif()

add_library( roc::rocfft-device ALIAS rocfft-device )
target_compile_features( rocfft-device PRIVATE
  cxx_static_assert cxx_nullptr cxx_auto_type )

if( NOT BUILD_SHARED_LIBS )
  target_link_libraries( rocfft-device INTERFACE hip::host )
endif()

if(HIP_PLATFORM STREQUAL "nvcc")
  target_compile_options( rocfft-device PRIVATE
    "-gencode arch=compute_75,code=sm_75"
    "-gencode arch=compute_70,code=sm_70"
    "-gencode arch=compute_60,code=sm_60" )
    target_compile_options( rocfft-device PRIVATE -Xcompiler -fPIC)
else()
  target_compile_options( rocfft-device PRIVATE -fno-gpu-rdc )
endif()

target_include_directories( rocfft-device
  PRIVATE $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
          $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/../include>
          $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/kernels>
          $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/generator>
          $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
          $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/library/include>
          $<BUILD_INTERFACE:${PROJECT_BINARY_DIR}/include>
)

rocm_set_soversion( rocfft-device ${rocfft_SOVERSION} )
set_target_properties( rocfft-device PROPERTIES CXX_EXTENSIONS NO )
set_target_properties( rocfft-device PROPERTIES RUNTIME_OUTPUT_DIRECTORY
  "${PROJECT_BINARY_DIR}/staging" )
set_target_properties( rocfft-device PROPERTIES DEBUG_POSTFIX "-d" )
set_target_properties( rocfft-device PROPERTIES CXX_STANDARD 14
  CXX_STANDARD_REQUIRED ON )

# Following Boost conventions of prefixing 'lib' on static built
# libraries, across all platforms
if( NOT BUILD_SHARED_LIBS )
  set_target_properties( rocfft-device PROPERTIES PREFIX "lib" )
endif( )

rocm_install_targets(
  TARGETS
  rocfft-device
  PREFIX
  rocfft )

rocm_install_symlink_subdir( rocfft )
