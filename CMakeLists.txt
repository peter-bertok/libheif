cmake_minimum_required (VERSION 3.16.3) # Oldest Ubuntu LTS (20.04 currently)

project(libheif LANGUAGES C CXX VERSION 1.17.1)

# compatibility_version is never allowed to be decreased for any specific SONAME.
# Libtool in the libheif-1.15.1 release had set it to 17.0.0, so we have to use this for the v1.x.y versions.
# With v2.0.0, we can reset this to more closely follow the real version name, i.e. "2.0.0".
# CMake 3.17.0 added "MACHO_COMPATIBILITY_VERSION", but we are currently stuck at cmake 3.16.3.
set(MACOS_COMPATIBLE_VERSION "17.0.0")

# https://cmake.org/cmake/help/v3.1/policy/CMP0054.html
cmake_policy(VERSION 3.0...3.22)
include(GNUInstallDirs)

# The version number.
set (PACKAGE_VERSION ${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}.${PROJECT_VERSION_PATCH})

# Check for unistd.h

include (${CMAKE_ROOT}/Modules/CheckIncludeFile.cmake)

CHECK_INCLUDE_FILE(unistd.h HAVE_UNISTD_H)

if (HAVE_UNISTD_H)
  add_definitions(-DHAVE_UNISTD_H)
endif()


if(NOT MSVC)
  add_definitions(-Wall)
  add_definitions(-Werror)
  add_definitions(-Wsign-compare)
  add_definitions(-Wconversion)
  add_definitions(-Wno-sign-conversion)
  add_definitions(-Wno-error=conversion)
  add_definitions(-Wno-error=unused-parameter)
  add_definitions(-Wno-error=deprecated-declarations)
  if (CMAKE_CXX_COMPILER_ID MATCHES "Clang")
      add_definitions(-Wno-error=tautological-compare)
      add_definitions(-Wno-error=tautological-constant-out-of-range-compare)
  endif ()
endif()

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

set(CMAKE_VISIBILITY_INLINES_HIDDEN 1)

set(CMAKE_POSITION_INDEPENDENT_CODE ON)

# Create the compile command database for clang by default
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

option(BUILD_SHARED_LIBS "Build shared libraries" ON)

include(CheckCXXCompilerFlag)
CHECK_CXX_COMPILER_FLAG(-Wno-error=potentially-evaluated-expression has_potentially_evaluated_expression)
if (has_potentially_evaluated_expression)
  add_definitions(-Wno-error=potentially-evaluated-expression)
endif()

LIST (APPEND CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake/modules")


# --- codec plugins

option(ENABLE_PLUGIN_LOADING "Support loading of plugins" ON)
set(PLUGIN_DIRECTORY "${CMAKE_INSTALL_FULL_LIBDIR}/libheif" CACHE STRING "Plugin install directory")

if (ENABLE_PLUGIN_LOADING)
    set(PLUGIN_LOADING_SUPPORTED_AND_ENABLED TRUE)
    install(DIRECTORY DESTINATION ${PLUGIN_DIRECTORY} DIRECTORY_PERMISSIONS
        OWNER_WRITE OWNER_READ OWNER_EXECUTE
        GROUP_READ GROUP_EXECUTE
        WORLD_READ WORLD_EXECUTE)
endif()

macro(plugin_option optionVariableName displayName defaultEnabled defaultPlugin)
    option(WITH_${optionVariableName} "Build ${displayName}" ${defaultEnabled})
    option(WITH_${optionVariableName}_PLUGIN "Build ${displayName} as a plugin" ${defaultPlugin})
endmacro()

macro(plugin_compilation_info optionVariableName detectionVariable displayName)
    if (${detectionVariable}_FOUND AND WITH_${optionVariableName}_PLUGIN AND PLUGIN_LOADING_SUPPORTED_AND_ENABLED)
        set(msg "+ separate plugin")
    elseif (${detectionVariable}_FOUND)
        set(msg "+ built-in")
    elseif (WITH_${optionVariableName})
        set(msg "- not found")
    else()
        set(msg "- disabled")
    endif ()

    string(LENGTH "${displayName}" len)
    math(EXPR fill "29 - ${len}")
    string(SUBSTRING "                                                " 0 ${fill} filler)
    message("${displayName}${filler}: ${msg}")
    unset(msg)
endmacro()

# libde265

plugin_option(LIBDE265 "libde265 HEVC decoder" ON OFF)
if (WITH_LIBDE265)
    find_package(LIBDE265)
endif()

# x265

plugin_option(X265 "x265 HEVC encoder" ON OFF)
if (WITH_X265)
    find_package(X265)
endif()

# kvazaar

plugin_option(KVAZAAR "kvazaar HEVC encoder" OFF OFF)
if (WITH_KVAZAAR)
    find_package(kvazaar)
    if ("${HAVE_KVAZAAR_ENABLE_LOGGING}")
        add_definitions(-DHAVE_KVAZAAR_ENABLE_LOGGING=1)
    else ()
        add_definitions(-DHAVE_KVAZAAR_ENABLE_LOGGING=0)
    endif ()
endif ()

# dav1d

plugin_option(DAV1D "Dav1d AV1 decoder" OFF ON)
if (WITH_DAV1D)
    find_package(DAV1D)
endif()

# aom

plugin_option(AOM_DECODER "AOM AV1 decoder" ON OFF)
plugin_option(AOM_ENCODER "AOM AV1 encoder" ON OFF)
if (WITH_AOM_ENCODER OR WITH_AOM_DECODER)
    find_package(AOM)
endif()

# svt

plugin_option(SvtEnc "SVT AV1 encoder" OFF ON)
if (WITH_SvtEnc)
    find_package(SvtEnc)
endif()

# rav1e

plugin_option(RAV1E "Rav1e AV1 encoder" OFF ON)
if (WITH_RAV1E)
    find_package(RAV1E)
endif()

# jpeg

plugin_option(JPEG_DECODER "JPEG decoder" OFF OFF)
plugin_option(JPEG_ENCODER "JPEG encoder" OFF OFF)
if (WITH_JPEG_ENCODER OR WITH_JPEG_DECODER)
    find_package(JPEG)
endif()

# openjpeg

plugin_option(OpenJPEG_ENCODER "OpenJPEG J2K encoder" OFF ON)
plugin_option(OpenJPEG_DECODER "OpenJPEG J2K decoder" OFF ON)
if (WITH_OpenJPEG_ENCODER OR WITH_OpenJPEG_DECODER)
    find_package(OpenJPEG)
endif()

# ffmpeg

plugin_option(FFMPEG_DECODER "FFMPEG HEVC decoder (HW accelerated)" OFF OFF)
if (WITH_FFMPEG_DECODER)
    find_package(FFMPEG COMPONENTS avcodec)
endif ()

# uncompressed

option(WITH_UNCOMPRESSED_CODEC " Support internal ISO/IEC 23001-17 uncompressed codec (experimental) " OFF)


# --- show codec compilation summary

message("=== Summary of compiled codecs ===")
plugin_compilation_info(LIBDE265 LIBDE265 "libde265 HEVC decoder")
plugin_compilation_info(FFMPEG_DECODER FFMPEG_avcodec "FFMPEG HEVC decoder (HW acc)")
plugin_compilation_info(X265 X265 "x265 HEVC encoder")
plugin_compilation_info(KVAZAAR KVAZAAR "Kvazaar HEVC encoder")
plugin_compilation_info(AOM_DECODER AOM_DECODER "AOM AV1 decoder")
plugin_compilation_info(AOM_ENCODER AOM_ENCODER "AOM AV1 encoder")
plugin_compilation_info(DAV1D DAV1D "Dav1d AV1 decoder")
plugin_compilation_info(SvtEnc SvtEnc "SVT AV1 encoder")
plugin_compilation_info(RAV1E RAV1E "Rav1e AV1 encoder")
plugin_compilation_info(JPEG_DECODER JPEG "JPEG decoder")
plugin_compilation_info(JPEG_ENCODER JPEG "JPEG encoder")
plugin_compilation_info(OpenJPEG_DECODER OpenJPEG "OpenJPEG J2K decoder")
plugin_compilation_info(OpenJPEG_ENCODER OpenJPEG "OpenJPEG J2K encoder")


# --- Libsharpyuv color space transforms

option(WITH_LIBSHARPYUV "Build libsharpyuv" ON)
if (WITH_LIBSHARPYUV)
    find_package(libsharpyuv)
endif ()
if (LIBSHARPYUV_FOUND)
    message("libsharpyuv: found")
elseif (WITH_${variableName})
    message("libsharpyuv: not found")
else()
    message("libsharpyuv: disabled")
endif ()

# --- Create libheif pkgconfig file

set(prefix ${CMAKE_INSTALL_PREFIX})
set(exec_prefix "\${prefix}")
if(IS_ABSOLUTE "${CMAKE_INSTALL_LIBDIR}")
    set(libdir "${CMAKE_INSTALL_LIBDIR}")
else()
    set(libdir "\${exec_prefix}/${CMAKE_INSTALL_LIBDIR}")
endif()
if(IS_ABSOLUTE "${CMAKE_INSTALL_INCLUDEDIR}")
    set(includedir "${CMAKE_INSTALL_INCLUDEDIR}")
else()
    set(includedir "\${prefix}/${CMAKE_INSTALL_INCLUDEDIR}")
endif()
if (LIBDE265_FOUND AND NOT (PLUGIN_LOADING_SUPPORTED_AND_ENABLED AND WITH_LIBDE265_PLUGIN))
    list(APPEND REQUIRES_PRIVATE "libde265")
endif()
if (X265_FOUND AND NOT (PLUGIN_LOADING_SUPPORTED_AND_ENABLED AND WITH_X265_PLUGIN))
    list(APPEND REQUIRES_PRIVATE "x265")
endif()
if ((AOM_DECODER_FOUND AND NOT (PLUGIN_LOADING_SUPPORTED_AND_ENABLED AND WITH_AOM_DECODER_PLUGIN))
        OR (AOM_ENCODER_FOUND AND NOT (PLUGIN_LOADING_SUPPORTED_AND_ENABLED AND WITH_AOM_ENCODER_PLUGIN)))
    list(APPEND REQUIRES_PRIVATE "aom")
endif()
if (FFMPEG_DECODER_FOUND AND NOT (PLUGIN_LOADING_SUPPORTED_AND_ENABLED AND WITH_FFMPEG_DECODER_PLUGIN))
    list(APPEND REQUIRES_PRIVATE "ffmpeg")
endif()
if (DAV1D_FOUND AND NOT (PLUGIN_LOADING_SUPPORTED_AND_ENABLED AND WITH_DAV1D_PLUGIN))
    list(APPEND REQUIRES_PRIVATE "dav1d")
endif()
if (RAV1E_FOUND AND NOT (PLUGIN_LOADING_SUPPORTED_AND_ENABLED AND WITH_RAV1E_PLUGIN))
    list(APPEND REQUIRES_PRIVATE "rav1e")
endif()
if (SvtEnc_FOUND AND NOT (PLUGIN_LOADING_SUPPORTED_AND_ENABLED AND WITH_SvtEnc_PLUGIN))
    list(APPEND REQUIRES_PRIVATE "SvtAv1Enc")
endif()
if (JPEG_FOUND AND ((WITH_JPEG_DECODER AND NOT (PLUGIN_LOADING_SUPPORTED_AND_ENABLED AND WITH_JPEG_DECODER_PLUGIN)) OR (WITH_JPEG_ENCODER AND NOT (PLUGIN_LOADING_SUPPORTED_AND_ENABLED AND WITH_JPEG_ENCODER_PLUGIN))))
    list(APPEND REQUIRES_PRIVATE "libjpeg")
endif()
if (OpenJPEG_FOUND AND ((WITH_OpenJPEG_DECODER AND NOT (PLUGIN_LOADING_SUPPORTED_AND_ENABLED AND WITH_OpenJPEG_DECODER_PLUGIN)) OR (WITH_OpenJPEG_ENCODER AND NOT (PLUGIN_LOADING_SUPPORTED_AND_ENABLED AND WITH_OpenJPEG_ENCODER_PLUGIN))))
    list(APPEND REQUIRES_PRIVATE "libopenjp2")
endif()
if (LIBSHARPYUV_FOUND)
    list(APPEND REQUIRES_PRIVATE "libsharpyuv")
endif()
list(JOIN REQUIRES_PRIVATE " " REQUIRES_PRIVATE)

include(CheckCXXSymbolExists)
check_cxx_symbol_exists(_LIBCPP_VERSION cstdlib HAVE_LIBCPP)
if(HAVE_LIBCPP)
  set(LIBS_PRIVATE "-lc++")
else()
  set(LIBS_PRIVATE "-lstdc++")
endif()

configure_file(libheif.pc.in ${CMAKE_CURRENT_BINARY_DIR}/libheif.pc @ONLY)

install(FILES ${CMAKE_CURRENT_BINARY_DIR}/libheif.pc
        DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)

# ---

option(WITH_EXAMPLES "Build examples" ON)
option(WITH_GDK_PIXBUF "Build gdk-pixbuf plugin" ON)

option(WITH_REDUCED_VISIBILITY "Reduced symbol visibility in library" ON)

option(WITH_DEFLATE_HEADER_COMPRESSION OFF)
option(ENABLE_MULTITHREADING_SUPPORT "Switch off for platforms without multithreading support" ON)
option(ENABLE_PARALLEL_TILE_DECODING "Will launch multiple decoders to decode tiles in parallel (requires ENABLE_MULTITHREADING_SUPPORT)" ON)

if (WITH_REDUCED_VISIBILITY)
    set(CMAKE_CXX_VISIBILITY_PRESET hidden)
else ()
    set(CMAKE_CXX_VISIBILITY_PRESET default)
endif ()

if(WITH_EXAMPLES)
    add_subdirectory (examples)
endif()

# --- API documentation

# check if Doxygen is installed
find_package(Doxygen)
if (DOXYGEN_FOUND)
    # set input and output files
    set(DOXYGEN_IN ${CMAKE_CURRENT_SOURCE_DIR}/libheif/Doxyfile.in)
    set(DOXYGEN_OUT ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile)

    # request to configure the file
    configure_file(${DOXYGEN_IN} ${DOXYGEN_OUT} @ONLY)
    message("Doxygen build started")

    # note the option ALL which allows to build the docs together with the application
    add_custom_target( doc_doxygen ALL
        COMMAND ${DOXYGEN_EXECUTABLE} ${DOXYGEN_OUT}
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
        COMMENT "Generating API documentation with Doxygen"
        VERBATIM )
else (DOXYGEN_FOUND)
  message("Doxygen tool needs to be installed to generate the API documentation")
endif (DOXYGEN_FOUND)

# --- Testing

option(BUILD_TESTING "" ON)
include(CTest)
if(BUILD_TESTING)
    # TODO: fix tests on windows.
    add_subdirectory (tests)
endif()


# --- Fuzzing

option(WITH_FUZZERS "Build the fuzzers (and no other executables)" OFF)
set(FUZZING_C_COMPILER "clang" CACHE STRING "C compiler to use for fuzzing")
set(FUZZING_CXX_COMPILER "clang++" CACHE STRING "C++ compiler to use for fuzzing")
set(FUZZING_COMPILE_OPTIONS "-fsanitize=fuzzer,address,shift,integer -fno-sanitize-recover=shift,integer" CACHE STRING "Compiler options for fuzzing")
set(FUZZING_LINKER_OPTIONS "" CACHE STRING "Additional linking options for fuzzing")

if (WITH_FUZZERS)
    set(CMAKE_C_COMPILER ${FUZZING_C_COMPILER})
    set(CMAKE_CXX_COMPILER ${FUZZING_CXX_COMPILER})
    message("Using compiler: ${CMAKE_CXX_COMPILER}")
    separate_arguments(FUZZING_COMPILE_OPTIONS UNIX_COMMAND "${FUZZING_COMPILE_OPTIONS}")
    separate_arguments(FUZZING_LINKER UNIX_COMMAND "${FUZZING_LINKER_OPTIONS}")
    add_compile_options(${FUZZING_COMPILE_OPTIONS})
    add_link_options(${FUZZING_COMPILE_OPTIONS} ${FUZZING_LINKER_OPTIONS})

    add_subdirectory(fuzzing)
endif()

if (CMAKE_CXX_COMPILER MATCHES "clang\\+\\+$")
    add_compile_options(-Wno-tautological-constant-out-of-range-compare)
endif()

add_subdirectory (libheif)

if (WITH_GDK_PIXBUF)
  add_subdirectory (gdk-pixbuf)
endif()

add_subdirectory (gnome)


# --- packaging (source code)

set(CPACK_VERBATIM_VARIABLES YES)
set(CPACK_SOURCE_GENERATOR TGZ)
set(CPACK_SOURCE_PACKAGE_FILE_NAME libheif-${PACKAGE_VERSION})
set(CPACK_SOURCE_IGNORE_FILES
/.git/
/.github/
/.gitignore$
/build/
/cmake-build.*/
/.deps/
/.idea/
/.clang-tidy
~$
/third-party/.*/ # only exclude the sub-directories, but keep the *.cmd files
/Testing/
/logos/
/Makefile$
/libtool$
/libheif.pc$
stamp-h1$
)
include(CPack)
