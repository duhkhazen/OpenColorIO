# SPDX-License-Identifier: BSD-3-Clause
# Copyright Contributors to the OpenColorIO Project.

cmake_minimum_required(VERSION 3.11)

project(OpenColorIO 
	VERSION 2.0.0
	LANGUAGES CXX C)

# "dev" for master or "" for any official release
set(OpenColorIO_VERSION_RELEASE_TYPE "dev")

enable_testing()

set(CMAKE_MODULE_PATH
    ${CMAKE_MODULE_PATH}
    ${CMAKE_SOURCE_DIR}/share/cmake/macros
    ${CMAKE_SOURCE_DIR}/share/cmake/modules
)
set(CMAKE_WARN_DEPRECATED ON)

if(NOT CMAKE_BUILD_TYPE)
	set(CMAKE_BUILD_TYPE Release)
endif()

###############################################################################
# C++ configuration

set(SUPPORTED_CXX_STANDARDS 11 14)
string(REPLACE ";" ", " SUPPORTED_CXX_STANDARDS_STR "${SUPPORTED_CXX_STANDARDS}")

if(CMAKE_CXX_STANDARD AND NOT CMAKE_CXX_STANDARD IN_LIST SUPPORTED_CXX_STANDARDS)
    message(FATAL_ERROR " CMAKE_CXX_STANDARD ${CMAKE_CXX_STANDARD} is unsupported. Supported standards are: ${SUPPORTED_CXX_STANDARDS_STR}")
elseif(NOT CMAKE_CXX_STANDARD)
    set(CMAKE_CXX_STANDARD 11)
endif()

# Disable fallback to other C++ version if standard is not supported.
set(CMAKE_CXX_STANDARD_REQUIRED ON)
# Disable any compiler specific C++ extensions.
set(CMAKE_CXX_EXTENSIONS OFF)

set(_BUILD_TYPE_DEBUG OFF)
if(CMAKE_BUILD_TYPE MATCHES "[Dd][Ee][Bb][Uu][Gg]")
    set(_BUILD_TYPE_DEBUG ON)
endif()
set(BUILD_TYPE_DEBUG ${_BUILD_TYPE_DEBUG} CACHE BOOL "Case-insensitive CMAKE_BUILD_TYPE Debug equality flag.")

###############################################################################
# Components to build

option(BUILD_SHARED_LIBS "Set to OFF to build static libraries" ON)
option(OCIO_BUILD_APPS "Set to OFF to disable command-line apps" ON)
option(OCIO_BUILD_NUKE "Specify whether to build nuke plugins" OFF)
option(OCIO_BUILD_DOCS "Specify whether to build documentation" OFF)
option(OCIO_BUILD_TESTS "Specify whether to build unittests" ON)
option(OCIO_BUILD_GPU_TESTS "Specify whether to build gpu unittests" ON)

option(OCIO_BUILD_PYTHON "Specify whether to build python bindings" ON)
option(OCIO_BUILD_JAVA "Specify whether to build java bindings" OFF)

option(OCIO_WARNING_AS_ERROR "Set build error level for CI testing" OFF)

###############################################################################
# Optimization / internal linking preferences

option(OCIO_USE_SSE "Specify whether to enable SSE CPU performance optimizations" ON)
option(OCIO_INLINES_HIDDEN "Specify whether to build with -fvisibility-inlines-hidden" ${UNIX})

###############################################################################
# Platform & Compiler dependent compilation flags

set(PLATFORM_COMPILE_FLAGS "")

if(MSVC)
	# Because our Exceptions derive from std::runtime_error we can safely disable warning 4275
	# /we4062 Enables warning in switch when an enumeration value is not explicitly handled.
	set(PLATFORM_COMPILE_FLAGS "${PLATFORM_COMPILE_FLAGS} /EHsc /wd4275 /DWIN32 /we4062")

else()
	# -Wswitch-enum Enables warning in switch when an enumeration value is not explicitly handled.
	set(PLATFORM_COMPILE_FLAGS "${PLATFORM_COMPILE_FLAGS} -Wall -Wswitch-enum")

	# TODO: OCIO being under active development, unused functions are fine for now.
	set(PLATFORM_COMPILE_FLAGS "${PLATFORM_COMPILE_FLAGS} -Wno-unused-function")

	# TODO: A C++17 specific warning when compiling in C++11 !!
	if("${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang")
		# Use of 'register' specifier must be removed for C++17 support.
		set(PLATFORM_COMPILE_FLAGS "${PLATFORM_COMPILE_FLAGS} -Wno-deprecated-register")
	endif()
endif()

if(OCIO_WARNING_AS_ERROR)
	# TODO: Vc++ compilation has still many tricky warnings difficult to fix.
	if(NOT MSVC)
		set(PLATFORM_COMPILE_FLAGS "${PLATFORM_COMPILE_FLAGS} -Werror")
	endif()
endif()

include(CheckSSEFeatures)
if(NOT HAVE_SSE2)
	message(STATUS "Disabling SSE optimizations, as the target doesn't support them")
	set(OCIO_USE_SSE OFF)
endif(NOT HAVE_SSE2)

###############################################################################
# External linking options

set(OCIO_INSTALL_EXT_PACKAGES "MISSING" CACHE STRING "Set the condition for Installing external dependencies")
set_property(CACHE OCIO_INSTALL_EXT_PACKAGES PROPERTY STRINGS "NONE" "MISSING" "ALL")

###############################################################################
# Versioning

if(NOT SOVERSION)
	set(SOVERSION "${OpenColorIO_VERSION_MAJOR}.${OpenColorIO_VERSION_MINOR}" CACHE STRING "Set the SO version in the SO name of the output library")
endif()

###############################################################################
# Namespace

if(NOT OCIO_NAMESPACE)
	set(OCIO_NAMESPACE OpenColorIO CACHE STRING "Specify the master OCIO C++ namespace: Options include OpenColorIO OpenColorIO_<YOURFACILITY> etc.")
endif()

###############################################################################
# Library name custom suffix
# This helps an application that needs to ship a dynamic library OCIO ensure
# that it has a unique name that won't conflict with one elsewhere on the
# system.

set (OCIO_LIBNAME_SUFFIX "" CACHE STRING
     "Specify a suffix to all libraries that are built")

###############################################################################
# Error checking

if(OCIO_BUILD_SHARED AND OCIO_BUILD_STATIC)
	message(FATAL_ERROR " Deprecated options OCIO_BUILD_SHARED and OCIO_BUILD_STATIC cannot both be used at once")
endif()

if(OCIO_BUILD_SHARED)
	message(DEPRECATION " Option OCIO_BUILD_SHARED is deprecated. Please use the cmake standard BUILD_SHARED_LIBS=ON (default ON)")
	if(NOT BUILD_SHARED_LIBS)
		set(BUILD_SHARED_LIBS ON)
	endif()
endif()

if(OCIO_BUILD_STATIC)
	message(DEPRECATION " Option OCIO_BUILD_STATIC is deprecated. Please use the cmake standard BUILD_SHARED_LIBS=OFF (default ON)")
	if(BUILD_SHARED_LIBS)
		set(BUILD_SHARED_LIBS OFF)
	endif()
endif()

###############################################################################
# Progress to other sources

add_subdirectory(tests)
add_subdirectory(include)
add_subdirectory(ext)
add_subdirectory(src)
if(OCIO_BUILD_DOCS)
	add_subdirectory(docs)
endif()
add_subdirectory(vendor)
