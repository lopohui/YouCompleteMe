# Copyright (C) 2011, 2012  Google Inc.
#
# This file is part of YouCompleteMe.
#
# YouCompleteMe is free software: you can redistribute it and/or modify
# it under the terms of the GNU General Public License as published by
# the Free Software Foundation, either version 3 of the License, or
# (at your option) any later version.
#
# YouCompleteMe is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
# GNU General Public License for more details.
#
# You should have received a copy of the GNU General Public License
# along with YouCompleteMe.  If not, see <http://www.gnu.org/licenses/>.

cmake_minimum_required( VERSION 2.8.7 )

project( ycm_core )

option( USE_DEV_FLAGS "Use compilation flags meant for YCM developers" OFF )
option( USE_CLANG_COMPLETER "Use Clang semantic completer for C/C++/ObjC" OFF )
option( USE_SYSTEM_LIBCLANG "Set to ON to use the system libclang library" OFF )
set( PATH_TO_LLVM_ROOT "" CACHE PATH "Path to the root of a LLVM+Clang binary distribution" )
set( EXTERNAL_LIBCLANG_PATH "" CACHE PATH "Path to the libclang library to use" )

if ( USE_CLANG_COMPLETER AND
     NOT USE_SYSTEM_LIBCLANG AND
     NOT PATH_TO_LLVM_ROOT AND
     NOT EXTERNAL_LIBCLANG_PATH )

  set( CLANG_VERSION "3.7.0" )

  if ( APPLE )
    set( CLANG_DIRNAME "clang+llvm-${CLANG_VERSION}-x86_64-apple-darwin" )
    set( CLANG_SHA256
         "057e56b445ac5dfbf0d5475badab374edebe8db5428001c8829340e40724b50d" )
    set( CLANG_FILENAME "${CLANG_DIRNAME}.tar.xz" )
  elseif ( WIN32 )
    if( 64_BIT_PLATFORM )
      set( CLANG_DIRNAME "LLVM-${CLANG_VERSION}-win64" )
      set( CLANG_SHA256
           "04a393cf73bc7416c9903aa08f47f15c77cf475b22a607ac4b7493a658a11a3f" )
    else()
      set( CLANG_DIRNAME "LLVM-${CLANG_VERSION}-win32" )
      set( CLANG_SHA256
           "75933480c8f8e3ae3cf9281494bf0057e9e4cfae4315f59c468a4e850a86f544" )
    endif()
    set( CLANG_FILENAME "${CLANG_DIRNAME}.exe" )
  else()
    if ( 64_BIT_PLATFORM )
      # We MUST support the latest Ubuntu LTS release!
      # At the time of writing, that's 14.04. Switch to next LTS ~6 months after
      # it comes out.
      set( CLANG_DIRNAME "clang+llvm-${CLANG_VERSION}-x86_64-linux-gnu-ubuntu-14.04" )
      set( CLANG_SHA256
           "093a94ff8982ae78461f0d2604c98f6b454c15e2ef768d34c235c6676c336460" )
      set( CLANG_FILENAME "${CLANG_DIRNAME}.tar.xz" )
    else()
      # Clang 3.3 is the last version with pre-built x86 binaries upstream.
      # That's too old now.
      message( FATAL_ERROR "No pre-built Clang ${CLANG_VERSION} binaries for 32 bit linux. "
        "You'll have to compile Clang ${CLANG_VERSION} from source. "
        "See the YCM docs for details on how to use a user-compiled libclang." )
    endif()
  endif()

  # Check if the Clang archive is already downloaded and its checksum is correct.
  # If this is not the case, remove it if needed and download it.
  set( CLANG_DOWNLOAD ON )
  set( CLANG_LOCAL_FILE
       "${CMAKE_SOURCE_DIR}/../clang_archives/${CLANG_FILENAME}" )

  if( EXISTS "${CLANG_LOCAL_FILE}" )
    file( SHA256 "${CLANG_LOCAL_FILE}" CLANG_LOCAL_SHA256 )

    if( "${CLANG_LOCAL_SHA256}" STREQUAL "${CLANG_SHA256}" )
      set( CLANG_DOWNLOAD OFF )
    else()
      file( REMOVE "${CLANG_LOCAL_FILE}" )
    endif()
  endif()

  if( CLANG_DOWNLOAD )
    message( "Downloading Clang ${CLANG_VERSION}" )

    set( CLANG_URL "http://llvm.org/releases/${CLANG_VERSION}" )
    file(
      DOWNLOAD "${CLANG_URL}/${CLANG_FILENAME}" "${CLANG_LOCAL_FILE}"
      SHOW_PROGRESS EXPECTED_HASH SHA256=${CLANG_SHA256}
    )
  else()
    message( "Using Clang archive: ${CLANG_LOCAL_FILE}" )
  endif()

  # Copy and extract the Clang archive in the building directory.
  file( COPY "${CLANG_LOCAL_FILE}" DESTINATION "${CMAKE_CURRENT_BINARY_DIR}/../" )
  if ( CLANG_FILENAME MATCHES ".+bz2" )
    execute_process( COMMAND tar -xjf ${CLANG_FILENAME} )
  elseif( CLANG_FILENAME MATCHES ".+xz" )
    execute_process( COMMAND tar -xJf ${CLANG_FILENAME} )
  elseif( CLANG_FILENAME MATCHES ".+exe" )
    # Find 7-zip executable in the PATH and using the registry
    find_program( 7Z_EXECUTABLE 7z PATHS
      [HKEY_LOCAL_MACHINE\\SOFTWARE\\7-Zip;Path] )
    if ( NOT 7Z_EXECUTABLE )
      message( FATAL_ERROR
        "7-zip is needed to extract the files from the Clang installer. "
        "Install it and try again." )
    endif()
    execute_process( COMMAND ${7Z_EXECUTABLE} x ${CLANG_FILENAME} OUTPUT_QUIET )
  else()
    execute_process( COMMAND tar -xzf ${CLANG_FILENAME} )
  endif()

  # We need to set PATH_TO_LLVM_ROOT. To do that, we first have to find the
  # folder name the archive produced. It isn't the archive base name.
  # On Windows, the find command is not available so we directly set the path.
  if ( WIN32 )
    # Since 7-zip release version 15.12, files are not anymore extracted in
    # $_OUTDIR folder but directly to the root. So, we check the existence of
    # $_OUTDIR to appropriately set PATH_TO_LLVM_ROOT.
    if ( EXISTS ${CMAKE_CURRENT_BINARY_DIR}/../$_OUTDIR )
      set( PATH_TO_LLVM_ROOT ${CMAKE_CURRENT_BINARY_DIR}/../$_OUTDIR )
    else()
      set( PATH_TO_LLVM_ROOT ${CMAKE_CURRENT_BINARY_DIR}/.. )
    endif()
  else()
    execute_process( COMMAND
      find ${CMAKE_CURRENT_BINARY_DIR}/.. -maxdepth 1 -type d -name clang*
      OUTPUT_VARIABLE PATH_TO_LLVM_ROOT
      OUTPUT_STRIP_TRAILING_WHITESPACE )
  endif()
endif()

if ( PATH_TO_LLVM_ROOT OR USE_SYSTEM_LIBCLANG OR EXTERNAL_LIBCLANG_PATH )
  set( USE_CLANG_COMPLETER TRUE )
endif()

if ( USE_CLANG_COMPLETER AND
     NOT PATH_TO_LLVM_ROOT AND
     NOT USE_SYSTEM_LIBCLANG AND
     NOT EXTERNAL_LIBCLANG_PATH )
  message( FATAL_ERROR
    "You have not specified which libclang to use. You have several options:\n"
    " 1. Set PATH_TO_LLVM_ROOT to a path to the root of a LLVM+Clang binary "
    "distribution. You can download such a binary distro from llvm.org. This "
    "is the recommended approach.\n"
    " 2. Set USE_SYSTEM_LIBCLANG to ON; this makes YCM search for the system "
    "version of libclang.\n"
    " 3. Set EXTERNAL_LIBCLANG_PATH to a path to whatever "
    "libclang.[so|dylib|dll] you wish to use.\n"
    "You HAVE to pick one option. See the docs for more information.")
endif()

if ( USE_CLANG_COMPLETER )
  message( "Using libclang to provide semantic completion for C/C++/ObjC" )
else()
  message( "NOT using libclang, no semantic completion for C/C++/ObjC will be "
           "available" )
endif()

if ( PATH_TO_LLVM_ROOT )
  set( CLANG_INCLUDES_DIR "${PATH_TO_LLVM_ROOT}/include" )
else()
  set( CLANG_INCLUDES_DIR "${CMAKE_SOURCE_DIR}/llvm/include" )
endif()

if ( NOT IS_ABSOLUTE "${CLANG_INCLUDES_DIR}" )
  get_filename_component(CLANG_INCLUDES_DIR
    "${CMAKE_BINARY_DIR}/${CLANG_INCLUDES_DIR}" ABSOLUTE)
endif()

if ( NOT EXTERNAL_LIBCLANG_PATH AND PATH_TO_LLVM_ROOT )
  if ( MINGW )
    set( LIBCLANG_SEARCH_PATH "${PATH_TO_LLVM_ROOT}/bin" )
  else()
    set( LIBCLANG_SEARCH_PATH "${PATH_TO_LLVM_ROOT}/lib" )
  endif()

  # Need TEMP because find_library does not work with an option variable
  find_library( TEMP NAMES clang libclang
                PATHS ${LIBCLANG_SEARCH_PATH}
                NO_DEFAULT_PATH )
  set( EXTERNAL_LIBCLANG_PATH ${TEMP} )
endif()

# This is a workaround for a CMake bug with include_directories(SYSTEM ...)
# on Mac OS X. Bug report: http://public.kitware.com/Bug/view.php?id=10837
if ( APPLE )
  set( CMAKE_INCLUDE_SYSTEM_FLAG_CXX "-isystem " )
endif()

if ( USE_SYSTEM_BOOST )
  find_package( Boost REQUIRED COMPONENTS python filesystem system regex thread )
else()
  set( Boost_INCLUDE_DIR ${BoostParts_SOURCE_DIR} )
  set( Boost_LIBRARIES BoostParts )
endif()

# The SYSTEM flag makes sure that -isystem[header path] is passed to the
# compiler instead of the standard -I[header path]. Headers included with
# -isystem do not generate warnings (and they shouldn't; e.g. boost warnings are
# just noise for us since we won't be changing them).
include_directories(
  SYSTEM
  ${Boost_INCLUDE_DIR}
  ${PYTHON_INCLUDE_DIRS}
  ${CLANG_INCLUDES_DIR}
  )

file( GLOB_RECURSE SERVER_SOURCES *.h *.cpp )

# The test sources are a part of a different target, so we remove them
# The CMakeFiles cpp file is picked up when the user creates an in-source build,
# and we don't want that. We also remove client-specific code
file( GLOB_RECURSE to_remove tests/*.h tests/*.cpp CMakeFiles/*.cpp *client* )

if( to_remove )
  list( REMOVE_ITEM SERVER_SOURCES ${to_remove} )
endif()

if ( USE_CLANG_COMPLETER )
  include_directories(
    ${CMAKE_CURRENT_SOURCE_DIR}
    "${CMAKE_CURRENT_SOURCE_DIR}/ClangCompleter" )
  add_definitions( -DUSE_CLANG_COMPLETER )
else()
  file( GLOB_RECURSE to_remove_clang ClangCompleter/*.h ClangCompleter/*.cpp )

  if( to_remove_clang )
    list( REMOVE_ITEM SERVER_SOURCES ${to_remove_clang} )
  endif()
endif()

#############################################################################

# One can use the system libclang.[so|dylib] like so:
#   cmake -DUSE_SYSTEM_LIBCLANG=1 [...]
# One can also explicitely pick the external libclang.[so|dylib] for use like so:
#   cmake -DEXTERNAL_LIBCLANG_PATH=/path/to/libclang.so [...]
# The final .so we build will then first look in the same dir in which it is
# located for libclang.so. This is provided by the rpath = $ORIGIN feature.

if ( EXTERNAL_LIBCLANG_PATH OR USE_SYSTEM_LIBCLANG )
  if ( USE_SYSTEM_LIBCLANG )
    if ( APPLE )
      set( ENV_LIB_PATHS ENV DYLD_LIBRARY_PATH )
    elseif ( UNIX )
      set( ENV_LIB_PATHS ENV LD_LIBRARY_PATH )
    elseif ( WIN32 )
      set( ENV_LIB_PATHS ENV PATH )
    else ()
      set( ENV_LIB_PATHS "" )
    endif()
    # On Debian-based systems, llvm installs into /usr/lib/llvm-x.y.
    file( GLOB SYS_LLVM_PATHS "/usr/lib/llvm*/lib" )
    # Need TEMP because find_library does not work with an option variable
    # On Debian-based systems only a symlink to libclang.so.1 is created
    find_library( TEMP
                  NAMES
                  clang
                  libclang.so.1
                  PATHS
                  ${ENV_LIB_PATHS}
                  /usr/lib
                  /usr/lib/llvm
                  ${SYS_LLVM_PATHS}
                  /Library/Developer/CommandLineTools/usr/lib
                  /Applications/Xcode.app/Contents/Developer/Toolchains/XcodeDefault.xctoolchain/usr/lib )
    set( EXTERNAL_LIBCLANG_PATH ${TEMP} )
  else()
    # For Macs, we do things differently; look further in this file.
    if ( NOT APPLE )
      # Setting this to true makes sure that libraries we build will have our rpath
      # set even without having to do "make install"
      set( CMAKE_BUILD_WITH_INSTALL_RPATH TRUE )
      set( CMAKE_INSTALL_RPATH "\$ORIGIN" )
    endif()
  endif()

  # In Clang 3.7, the library target is a symlink of the soversion one.
  # Since it will be copied in the project folder, we need the symlinked
  # library.
  get_filename_component( LIBCLANG_TARGET "${EXTERNAL_LIBCLANG_PATH}" REALPATH )
  message(
    "Using external libclang: ${LIBCLANG_TARGET}" )
else()
  set( LIBCLANG_TARGET )
endif()

if ( EXTRA_RPATH )
  set( CMAKE_INSTALL_RPATH "${EXTRA_RPATH}:${CMAKE_INSTALL_RPATH}" )
endif()

# Needed on Linux machines, but not on Macs and OpenBSD
if ( UNIX AND NOT ( APPLE OR SYSTEM_IS_OPENBSD ) )
  set( EXTRA_LIBS rt )
endif()

#############################################################################

add_library( ${PROJECT_NAME} SHARED
             ${SERVER_SOURCES}
           )

target_link_libraries( ${PROJECT_NAME}
                       ${Boost_LIBRARIES}
                       ${PYTHON_LIBRARIES}
                       ${LIBCLANG_TARGET}
                       ${EXTRA_LIBS}
                     )

if( LIBCLANG_TARGET )
  # When building with MSVC, we need to copy libclang.dll instead of libclang.lib
  if( MSVC )
    add_custom_command(
      TARGET ${PROJECT_NAME}
      POST_BUILD
      COMMAND ${CMAKE_COMMAND} -E copy "${PATH_TO_LLVM_ROOT}/bin/libclang.dll" "$<TARGET_FILE_DIR:${PROJECT_NAME}>"
    )
  else()
    add_custom_command(
      TARGET ${PROJECT_NAME}
      POST_BUILD
      COMMAND ${CMAKE_COMMAND} -E copy "${LIBCLANG_TARGET}" "$<TARGET_FILE_DIR:${PROJECT_NAME}>"
    )
  endif()
endif()


#############################################################################

# Things are a bit different on Macs when using an external libclang.dylib; here
# we want to make sure we use @loader_path/libclang.dylib instead of
# @rpath/libclang.dylib in the final ycm_core.so. If we use the
# @rpath version, then it may load the system libclang which the user
# explicitely does not want (otherwise the user would specify
# USE_SYSTEM_LIBCLANG). If we hardcode with `@loader_path/libclang.dylib`,
# it guarantees to search `@loader_path` for libclang.dylib first and use it
# only. So with `@loader_path`, we make sure that only the libclang.dylib
# present in the same directory as our ycm_core.so is used.
# And the extra `rpath` is helpful to find libclang.dylib's dependencies
# if it is built with `--enable-shared` flag.
if ( EXTERNAL_LIBCLANG_PATH AND APPLE )
  get_filename_component(EXTRA_LIB_PATH ${EXTERNAL_LIBCLANG_PATH} DIRECTORY)

  set_target_properties(${PROJECT_NAME}
    PROPERTIES LINK_FLAGS "-Wl,-rpath,${EXTRA_LIB_PATH}")

  add_custom_command( TARGET ${PROJECT_NAME}
                      POST_BUILD
                      COMMAND install_name_tool
                      "-change"
                      "@rpath/libclang.dylib"
                      "@loader_path/libclang.dylib"
                      "$<TARGET_FILE:${PROJECT_NAME}>"
                    )
endif()


#############################################################################

# We don't want the "lib" prefix, it can screw up python when it tries to search
# for our module
set_target_properties( ${PROJECT_NAME} PROPERTIES PREFIX "")

if ( WIN32 OR CYGWIN )
  # DLL platforms put dlls in the RUNTIME_OUTPUT_DIRECTORY
  # First for the generic no-config case (e.g. with mingw)
  set_target_properties( ${PROJECT_NAME} PROPERTIES
    RUNTIME_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/../.. )
  # Second, for multi-config builds (e.g. msvc)
  foreach( OUTPUTCONFIG ${CMAKE_CONFIGURATION_TYPES} )
    string( TOUPPER ${OUTPUTCONFIG} OUTPUTCONFIG )
    set_target_properties( ${PROJECT_NAME} PROPERTIES
      RUNTIME_OUTPUT_DIRECTORY_${OUTPUTCONFIG} ${PROJECT_SOURCE_DIR}/../.. )
  endforeach()

  if ( WIN32 )
    # This is the extension for compiled Python modules on Windows
    set_target_properties( ${PROJECT_NAME} PROPERTIES SUFFIX ".pyd")
  elseif ( CYGWIN )
    # This is the extension for compiled Python modules in Cygwin
    set_target_properties( ${PROJECT_NAME} PROPERTIES SUFFIX ".dll")
  endif()
else()
  # Even on macs, we want a .so extension instead of a .dylib which is what
  # cmake would give us by default. Python won't recognize a .dylib as a module,
  # but it will recognize a .so
  set_target_properties( ${PROJECT_NAME} PROPERTIES SUFFIX ".so")
endif()

set_target_properties( ${PROJECT_NAME} PROPERTIES
  LIBRARY_OUTPUT_DIRECTORY ${PROJECT_SOURCE_DIR}/../.. )

#############################################################################


# For some reason, Xcode is too dumb to understand the -isystem flag and thus
# borks on warnings in Boost.
if ( USE_DEV_FLAGS AND ( CMAKE_COMPILER_IS_GNUCXX OR COMPILER_IS_CLANG ) AND
     NOT CMAKE_GENERATOR_IS_XCODE )
  # We want all warnings, and warnings should be treated as errors
  set( CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wextra -Werror" )
endif()

#############################################################################

# We want warnings if we accidentally use C++11 features
# We can't use this warning on FreeBSD because std headers on that OS are dumb.
# See here: https://github.com/Valloric/YouCompleteMe/issues/260
if ( USE_DEV_FLAGS AND COMPILER_IS_CLANG AND NOT CMAKE_GENERATOR_IS_XCODE AND
     NOT SYSTEM_IS_FREEBSD )
  set( CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wc++98-compat" )
endif()

#############################################################################

if( SYSTEM_IS_SUNOS )
  # SunOS needs this setting for thread support
  set( CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pthreads" )
endif()

if( SYSTEM_IS_OPENBSD OR SYSTEM_IS_FREEBSD )
  set( CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -pthread" )
endif()

add_subdirectory( tests )
