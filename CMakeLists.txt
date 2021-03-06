# Copyright (c) Dean Michael Berris 2010.
# Copyright (c) Google, Inc. 2012.
# Copyright (c) Glyn Matthews 2013.
# Distributed under the Boost Software License, Version 1.0.
# (See accompanying file LICENSE_1_0.txt or copy at
# http://www.boost.org/LICENSE_1_0.txt)

cmake_minimum_required(VERSION 2.8)
project(CPP-NETLIB)

option( CPP-NETLIB_BUILD_SHARED_LIBS "Build cpp-netlib as shared libraries." OFF )
option( CPP-NETLIB_BUILD_SINGLE_LIB "Build cpp-netlib into a single library" OFF )
option( CPP-NETLIB_BUILD_TESTS "Build the unit tests." ON )
option( CPP-NETLIB_BUILD_EXAMPLES "Build the examples using cpp-netlib." ON )
option( CPP-NETLIB_ALWAYS_LOGGING "Allow cpp-netlib to log debug messages even in non-debug mode." OFF )
option( CPP-NETLIB_DISABLE_LOGGING "Disable logging definitely, no logging code will be generated or compiled." OFF )
option( CPP-NETLIB_DISABLE_LIBCXX "Disable using libc++ when compiling with clang." OFF )
option( CPP-NETLIB_DISABLE_FEATURE_TESTS "Disable the feature tests (which may use a network connection)." OFF )

set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR})
find_package( ICU )

if(CPP-NETLIB_BUILD_SHARED_LIBS OR BUILD_SHARED_LIBS)
  set(Boost_USE_STATIC_LIBS OFF)
  set(CPP-NETLIB_BUILD_SHARED_LIBS ON)
  set(BUILD_SHARED_LIBS ON)
else()
  set(Boost_USE_STATIC_LIBS ON)
  set(CPP-NETLIB_BUILD_SHARED_LIBS OFF)
  set(BUILD_SHARED_LIBS OFF)
endif()
set(Boost_USE_MULTITHREADED ON)
set(Boost_COMPONENTS system regex filesystem )
find_package( Boost 1.53 REQUIRED ${Boost_COMPONENTS} )
find_package( OpenSSL )
find_package( Threads )
set(CMAKE_VERBOSE_MAKEFILE true)

if(CMAKE_BUILD_TYPE MATCHES Debug)
  add_definitions(-DNETWORK_DEBUG)
endif()

if (OPENSSL_FOUND)
  add_definitions(-DNETWORK_ENABLE_HTTPS)
endif()

if (${CMAKE_CXX_COMPILER_ID} MATCHES GNU)
  INCLUDE(CheckCXXCompilerFlag)
  CHECK_CXX_COMPILER_FLAG(-std=c++11 HAVE_STD11)
  if (HAVE_STD11)
    #set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -Wall")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
  else()
    message(FATAL_ERROR "No advanced standard C++ support (-std=c++11 not defined).")
  endif()
elseif(${CMAKE_CXX_COMPILER_ID} MATCHES Clang)
  INCLUDE(CheckCXXCompilerFlag)
  CHECK_CXX_COMPILER_FLAG(-std=c++11 HAVE_STD11)
  if (HAVE_STD11)
    if (CPP-NETLIB_DISABLE_LIBCXX)
      #set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -Wall")
      set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
    else()
      #set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -stdlib=libc++ -Wall")
      set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -stdlib=libc++")
      set(CMAKE_CXX_LINK_FLAGS "${CMAKE_CXX_LINK_FLAGS} -stdlib=libc++")
    endif()
  else()
    message(FATAL_ERROR "No C++11 support for Clang version. Please upgrade Clang to a version supporting C++11.")
  endif()
  add_definitions(
  -DASIO_HAS_MOVE -DASIO_HAS_VARIADIC_TEMPLATES -DASIO_HAS_STD_SYSTEM_ERROR
  -DASIO_ERROR_CATEGORY_NOEXCEPT=noexcept -DASIO_HAS_STD_ARRAY
  -DASIO_HAS_STD_SHARED_PTR -DASIO_HAS_STD_ATOMIC -DASIO_HAS_STD_CHRONO
  -DASIO_HAS_STD_ADDRESSOFF -DASIO_HAS_STD_FUNCTION -DASIO_HAS_STD_TYPE_TRAITS)
endif()

message(STATUS "C++ Compiler ID: ${CMAKE_CXX_COMPILER_ID}")
message(STATUS "C++ Flags: ${CMAKE_CXX_FLAGS} link flags: ${CMAKE_CXX_LINK_FLAGS}")

if (MSVC)
  add_definitions(-D_SCL_SECURE_NO_WARNINGS)
endif(MSVC)
if (WIN32)
  add_definitions(-D_WIN32_WINNT=0x0501)
endif(WIN32)
include_directories(${Boost_INCLUDE_DIRS})

message(STATUS "CPP-NETLIB options selected:")
message(STATUS "  CPP-NETLIB_BUILD_SHARED_LIBS:      ${CPP-NETLIB_BUILD_SHARED_LIBS}\t(Build cpp-netlib as shared libraries: OFF, ON)")
message(STATUS "  CPP-NETLIB_BUILD_SINGLE_LIB:       ${CPP-NETLIB_BUILD_SINGLE_LIB}\t(Build cpp-netlib into a single library: OFF, ON)")
message(STATUS "  CPP-NETLIB_BUILD_TESTS:            ${CPP-NETLIB_BUILD_TESTS}\t(Build the unit tests: ON, OFF)")
message(STATUS "  CPP-NETLIB_BUILD_EXAMPLES:         ${CPP-NETLIB_BUILD_EXAMPLES}\t(Build the examples using cpp-netlib: ON, OFF)")
message(STATUS "  CPP-NETLIB_ALWAYS_LOGGING:         ${CPP-NETLIB_ALWAYS_LOGGING}\t(Allow cpp-netlib to log debug messages even in non-debug mode: ON, OFF)")
message(STATUS "  CPP-NETLIB_DISABLE_LOGGING:        ${CPP-NETLIB_DISABLE_LOGGING}\t(Disable logging definitely, no logging code will be generated or compiled: ON, OFF)")
message(STATUS "  CPP-NETLIB_DISABLE_LIBCXX:         ${CPP-NETLIB_DISABLE_LIBCXX}\t(Disable using libc++ when building with clang: ON, OFF)")
message(STATUS "  CPP-NETLIB_DISABLE_FEATURE_TESTS:  ${CPP-NETLIB_DISABLE_FEATURE_TESTS}\t(Disable the feature tests (which may use a network connection): ON, OFF)")
message(STATUS "CMake build options selected:")

# Takes one or more lists as parameter and prepends the prefix to every element.
function(prependToElements prefix)
  foreach(list ${ARGN} ) # iterate over lists
    set(newlist "")
    foreach(ele ${${list}} ) # iterate over the elements
      set(newlist ${newlist} "${prefix}${ele}")
    endforeach(ele)
    set(${list} ${newlist} PARENT_SCOPE) # replace the original list
  endforeach(list)
endfunction(prependToElements)

############################################################################
#
# The code following this point is for the new directory structure
#

if(CPP-NETLIB_BUILD_TESTS)
  enable_testing()
  if(MSVC11)
    add_definitions(-D_VARIADIC_MAX=10)
  endif(MSVC11)

  if(MSVC)
    set(gtest_force_shared_crt ON CACHE BOOL "Override gtest option.")
  endif(MSVC)

  # gmock automatically searches for gtest
  add_subdirectory(deps/gmock)

  set(GTEST_ROOT ${CPP-NETLIB_SOURCE_DIR}/deps/gtest)
  set(GTEST_FOUND ON)
  set(GTEST_INCLUDE_DIRS ${GTEST_ROOT}/include)
  set(GTEST_LIBRARY gtest)
  set(GTEST_MAIN_LIBRARY gtest_main)
  set(GTEST_BOTH_LIBRARIES ${GTEST_LIBRARY} ${GTEST_MAIN_LIBRARY})

  set(GMOCK_ROOT ${CPP-NETLIB_SOURCE_DIR}/deps/gmock)
  set(GMOCK_FOUND ON)
  set(GMOCK_INCLUDE_DIRS ${GTEST_INCLUDE_DIRS} ${GMOCK_ROOT}/include)
  set(GMOCK_LIBRARY ${GTEST_LIBRARIES} gmock)
  set(GMOCK_MAIN_LIBRARY gmock_main)
  set(GMOCK_BOTH_LIBRARIES ${GMOCK_LIBRARY} ${GMOCK_MAIN_LIBRARY})

  set(IGLOO_ROOT ${CPP-NETLIB_SOURCE_DIR}/deps/igloo)
  set(IGLOO_FOUND ON)
  set(IGLOO_INCLUDE_DIR ${IGLOO_ROOT})
endif()

# Documentation
find_package(Doxygen)
if (DOXYGEN_FOUND)
  configure_file(${CMAKE_CURRENT_SOURCE_DIR}/Doxyfile.in ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile @ONLY)
  add_custom_target(doc
    ${DOXYGEN_EXECUTABLE} ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile
    WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
    COMMENT "Generating API documentation with Doxygen" VERBATIM)
  set(CPP-NETLIB_GENERATE_DOCS ON)
endif(DOXYGEN_FOUND)

if(CPP-NETLIB_BUILD_SINGLE_LIB)
  include_directories(
    ${CMAKE_CURRENT_SOURCE_DIR}/config/src
    ${CMAKE_CURRENT_SOURCE_DIR}/concurrency/src
    ${CMAKE_CURRENT_SOURCE_DIR}/http/src
    ${CMAKE_CURRENT_SOURCE_DIR}/logging/src
    ${CMAKE_CURRENT_SOURCE_DIR}/message/src
    ${CMAKE_CURRENT_SOURCE_DIR}/uri/src
    ${CMAKE_CURRENT_SOURCE_DIR}
  )
endif()

add_subdirectory(uri)
add_subdirectory(message)
add_subdirectory(logging)
add_subdirectory(http)
add_subdirectory(concurrency)
#add_subdirectory(mime)
if(CPP-NETLIB_BUILD_EXAMPLES)
  add_subdirectory(contrib/http_examples)
endif()

if(CPP-NETLIB_BUILD_SINGLE_LIB)
  add_library(cppnetlib
    ${Uri_SRCS}
    ${CPP-NETLIB_MESSAGE_SRCS}
    ${CPP-NETLIB_MESSAGE_DIRECTIVES_SRCS}
    ${CPP-NETLIB_MESSAGE_WRAPPERS_SRCS}
    ${CPP-NETLIB_LOGGING_SRCS}
    ${CPP-NETLIB_HTTP_CLIENT_SRCS}
    ${CPP-NETLIB_HTTP_CLIENT_CONNECTIONS_SRCS}
    ${CPP-NETLIB_HTTP_SERVER_SRCS}
    ${CPP-NETLIB_HTTP_MESSAGE_SRCS}
    ${CPP-NETLIB_HTTP_MESSAGE_WRAPPERS_SRCS}
    ${CPP-NETLIB_CONSTANTS_SRCS}
    ${CPP-NETLIB_CONCURRENCY_SRCS}
  )
  target_link_libraries(cppnetlib ${Boost_LIBRARIES})
endif()
