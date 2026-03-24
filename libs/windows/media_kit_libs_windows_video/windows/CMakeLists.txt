# This file is a part of media_kit (https://github.com/media-kit/media-kit).
#
# Copyright © 2021 & onwards, Hitesh Kumar Saini <saini123hitesh@gmail.com>.
# All rights reserved.
# Use of this source code is governed by MIT license that can be found in the LICENSE file.

cmake_minimum_required(VERSION 3.14)
cmake_policy(SET CMP0175 NEW)

# This option is read by the other packages which are part of package:media_kit.
option(MEDIA_KIT_LIBS_AVAILABLE "package:media_kit libraries are available." ON)

set(PROJECT_NAME "media_kit_libs_windows_video")
project(${PROJECT_NAME} LANGUAGES CXX)

# Deal with MSVC incompatiblity
add_compile_definitions(_DISABLE_CONSTEXPR_MUTEX_CONSTRUCTOR)

# ------------------------------------------------------------------------------
function(download_and_verify url sha256 locationForArchive)
  # Check if the archive exists.
  if(EXISTS "${locationForArchive}")
    file(SHA256 "${locationForArchive}" ARCHIVE_SHA256)

    # If SHA256 doesn't match, delete the archive to download again.
    if(NOT sha256 STREQUAL ARCHIVE_SHA256)
      file(REMOVE "${locationForArchive}")
      message(STATUS "SHA256 mismatch. File deleted.")
    endif()
  endif()

  # Download the archive if it doesn't exist.
  if(NOT EXISTS "${locationForArchive}")
    message(STATUS "Downloading archive from ${url}...")
    file(DOWNLOAD "${url}" "${locationForArchive}")
    message(STATUS "Downloaded archive to ${locationForArchive}.")

    # Verify SHA256 of the newly downloaded file.
    file(SHA256 "${locationForArchive}" ARCHIVE_SHA256)

    if(sha256 STREQUAL ARCHIVE_SHA256)
      message(STATUS "${locationForArchive} Verification successful.")
    else()
      message(FATAL_ERROR "${locationForArchive} Integrity check failed, please try to re-build project again.")
    endif()
  endif()
endfunction()

function(check_directory_exists_and_not_empty dir result_var)
  # Check if the directory exists
  if(EXISTS "${dir}")
    # Check if the directory is not empty
    file(GLOB dir_contents "${dir}/*")

    if(dir_contents)
      set(${result_var} TRUE PARENT_SCOPE)
    else()
      set(${result_var} FALSE PARENT_SCOPE)
      message(STATUS "Directory ${dir} exists but is empty!")
    endif()
  else()
    set(${result_var} FALSE PARENT_SCOPE)
    message(STATUS "Directory ${dir} does not exist!")
  endif()
endfunction()

# Detect architecture and set appropriate libmpv archive
# FLUTTER_TARGET_PLATFORM is set by Flutter build system (windows-x64 or windows-arm64)
if(${FLUTTER_TARGET_PLATFORM} STREQUAL "windows-x64")
  set(LIBMPV_ARCH "x86_64")
  set(LIBMPV "mpv-dev-x86_64-20250930-git-05656cd.7z")
  set(LIBMPV_SHA256 "2a44d51385da539da803f1b5f5053fd8")
else()
  # windows-arm64
  set(LIBMPV_ARCH "aarch64")
  set(LIBMPV "mpv-dev-aarch64-20251004-git-17a3ac4.7z")
  set(LIBMPV_SHA256 "d3c2d7750ae19780f85eaa605d4e4b9bbd9f8336f020fc12d050958006f19333")
endif()

# Download URL for the libmpv archive
set(LIBMPV_URL "https://github.com/bggRGjQaUbCoE/mpv-winbuild-cmake/releases/download/20251004/${LIBMPV}")

# Download location of the libmpv archive.
set(LIBMPV_ARCHIVE "${CMAKE_BINARY_DIR}/${LIBMPV}")
set(LIBMPV_SRC "${CMAKE_BINARY_DIR}/libmpv")

download_and_verify(
  ${LIBMPV_URL}
  ${LIBMPV_SHA256}
  ${LIBMPV_ARCHIVE}
)

check_directory_exists_and_not_empty(${LIBMPV_SRC} LIBMPV_SRC_VALID)

# Find 7zip for extraction (required for .7z files)
find_program(SEVENZ_EXECUTABLE NAMES 7z 7za)

# Extract the libmpv archive.
if(NOT LIBMPV_SRC_VALID)
  message(STATUS "Extracting ${LIBMPV}...")
  make_directory("${LIBMPV_SRC}")
  add_custom_target("${PROJECT_NAME}_LIBMPV_EXTRACT" ALL)

  if(SEVENZ_EXECUTABLE)
    message(STATUS "Using 7zip for extraction: ${SEVENZ_EXECUTABLE}")
    add_custom_command(
      TARGET "${PROJECT_NAME}_LIBMPV_EXTRACT"
      POST_BUILD
      COMMAND "${SEVENZ_EXECUTABLE}" x -y "\"${LIBMPV_ARCHIVE}\""
      COMMAND xcopy "\"${LIBMPV_SRC}/include/mpv\"" "\"${LIBMPV_SRC}/mpv\"" /E /H /C /I
      COMMAND rmdir "\"${LIBMPV_SRC}/include\"" /S /Q
      COMMAND ren "\"${LIBMPV_SRC}/mpv\"" "\"include\""
      WORKING_DIRECTORY "${LIBMPV_SRC}"
    )
  else()
    message(FATAL_ERROR "7zip not found. Please install 7-Zip to extract .7z archives. Download from: https://www.7-zip.org/")
  endif()
endif()

# ------------------------------------------------------------------------------

# ANGLE archive - only for x64 (ARM64 uses native D3D11)
if(${FLUTTER_TARGET_PLATFORM} STREQUAL "windows-x64")
  set(ANGLE "ANGLE.7z")
  set(ANGLE_URL "https://github.com/alexmercerind/flutter-windows-ANGLE-OpenGL-ES/releases/download/v1.0.1/ANGLE.7z")
  set(ANGLE_SHA256 "e866f13e8d552348058afaafe869b1ed")

  set(ANGLE_ARCHIVE "${CMAKE_BINARY_DIR}/${ANGLE}")
  set(ANGLE_SRC "${CMAKE_BINARY_DIR}/ANGLE")

  download_and_verify(
    ${ANGLE_URL}
    ${ANGLE_SHA256}
    ${ANGLE_ARCHIVE}
  )

  check_directory_exists_and_not_empty(${ANGLE_SRC} ANGLE_SRC_VALID)

  if(NOT ANGLE_SRC_VALID)
    message(STATUS "Extracting ${ANGLE}...")
    make_directory("${ANGLE_SRC}")
    add_custom_target("${PROJECT_NAME}_ANGLE_EXTRACT" ALL)

    if(SEVENZ_EXECUTABLE)
      add_custom_command(
        TARGET "${PROJECT_NAME}_ANGLE_EXTRACT"
        POST_BUILD
        COMMAND "${SEVENZ_EXECUTABLE}" x -y "\"${ANGLE_ARCHIVE}\""
        WORKING_DIRECTORY "${ANGLE_SRC}"
      )
    endif()
  endif()
endif()

# ------------------------------------------------------------------------------
set(PLUGIN_NAME "media_kit_libs_windows_video_plugin")

add_library(
  ${PLUGIN_NAME} SHARED
  "include/media_kit_libs_windows_video/media_kit_libs_windows_video_plugin_c_api.h"
  "media_kit_libs_windows_video_plugin_c_api.cpp"
)

apply_standard_settings(${PLUGIN_NAME})

set_target_properties(
  ${PLUGIN_NAME}
  PROPERTIES
  CXX_VISIBILITY_PRESET
  hidden
)
target_compile_definitions(
  ${PLUGIN_NAME}
  PRIVATE
  FLUTTER_PLUGIN_IMPL
)

target_include_directories(
  ${PLUGIN_NAME} INTERFACE
  "${CMAKE_CURRENT_SOURCE_DIR}/include"
)
target_link_libraries(
  ${PLUGIN_NAME}
  PRIVATE
  flutter
  flutter_wrapper_plugin
)

if(${FLUTTER_TARGET_PLATFORM} STREQUAL "windows-x64")
  set(
    media_kit_libs_windows_video_bundled_libraries
    "${LIBMPV_SRC}/libmpv-2.dll"
    "${ANGLE_SRC}/d3dcompiler_47.dll"
    "${ANGLE_SRC}/libEGL.dll"
    "${ANGLE_SRC}/libGLESv2.dll"
    "${ANGLE_SRC}/vk_swiftshader.dll"
    "${ANGLE_SRC}/vulkan-1.dll"
    "${ANGLE_SRC}/zlib.dll"
    PARENT_SCOPE
  )
else()
  # ARM64 - no ANGLE needed, native D3D11
  set(
    media_kit_libs_windows_video_bundled_libraries
    "${LIBMPV_SRC}/libmpv-2.dll"
    PARENT_SCOPE
  )
endif()