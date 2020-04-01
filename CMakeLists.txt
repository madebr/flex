cmake_minimum_required(VERSION 3.0.2 FATAL_ERROR)

list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_LIST_DIR}/cmake")

if(NOT CMAKE_BUILD_TYPE)
    message(STATUS "No build type specified, defaulting to Release.")
    set(CMAKE_BUILD_TYPE "Release" CACHE STRING "Choose the type of build, options are: DEBUG, RELEASE, or MINSIZEREL." FORCE)
endif()

file(READ "configure.ac" configure_ac_contents)
string(REGEX MATCH "AC_INIT\\(\\[[a-zA-Z ]+\\],\\[([0-9.]+)\\]," flex_version_match "${configure_ac_contents}")
if(NOT flex_version_match)
    message(FATAL_ERROR "Could not parse flex version from configure.ac")
endif()
set(BISON_VERSION "${CMAKE_MATCH_1}")

project(flex VERSION "${BISON_VERSION}" LANGUAGES C)

include(GNUInstallDirs)
set(CMAKE_INSTALL_CMAKEDIR "${CMAKE_INSTALL_LIBDIR}/cmake/flex" CACHE PATH "CMake scripts")

# Create config.h
include(make_config_h)

# Build options
option(ENABLE_NLS "Enable NLS" FALSE)
if(ENABLE_NLS)
  add_definitions(ENABLE_NLS YYENABLE_NLS)
endif()
option(ENABLE_C_ALLOCA "Enable alloca.c" FALSE)
option(ENABLE_MEM_SHIM "Replacement malloc, realloc" FALSE)

# Locate M4 path if not set
find_program(M4_BIN
  NAMES gm4 gnum4 m4
)
message(STATUS "Using M4: ${M4_BIN}")

# compiler specifics
if (CMAKE_C_COMPILER_ID MATCHES "(GNU)|(Clang)")
  add_definitions(_GNU_SOURCE)
elseif (CMAKE_C_COMPILER_ID MATCHES "MSVC")
  add_definitions(_CONSOLE _MBCS _CRT_SECURE_NO_WARNINGS)
endif()

# generate parse.c
find_package(BISON REQUIRED)
bison_target(parse src/parse.y gen/parse.c)

# generate scan.c
find_package(FLEX)
flex_target(scan src/scan.l gen/scan.c)

# generate skel.c
string(REPLACE ";" "" STRIPPED_VERSION "${VERSION}")
find_package(UnixCommands)
find_program(SH_BIN NAMES sh zsh bash)
if(SH_BIN)
  add_custom_command(
    OUTPUT ${CMAKE_BINARY_DIR}/gen/skel.c
    COMMAND ${SH_BIN} -c "${CMAKE_SOURCE_DIR}/src/mkskel.sh "${CMAKE_SOURCE_DIR}/src" "${M4_BIN}" "${PACKAGE_VERSION}">skel.c"
    DEPENDS "${CMAKE_SOURCE_DIR}/src/mkskel.sh"
    WORKING_DIRECTORY ${CMAKE_BINARY_DIR}/gen
    COMMENT "[m4][flex.skl] Creating skel.c"
  )
  # TODO make this step target generic via CMake script
endif()

set(GENERATED_SOURCES
  ${BISON_parse_OUTPUTS}
  ${FLEX_scan_OUTPUTS}
  ${CMAKE_BINARY_DIR}/gen/skel.c
)

# include files
include_directories(
  "$<BUILD_INTERFACE:src>"
  "$<BUILD_INTERFACE:${CMAKE_BINARY_DIR}/gen>"
  "$<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}"
)

# source files
set(sources
  src/buf.c
  src/ccl.c
  src/dfa.c
  src/ecs.c
  src/scanflags.c
  src/filter.c
  src/flexdef.h
  src/gen.c
  src/misc.c
  src/nfa.c
  src/options.c
  src/options.h
  src/regex.c
  src/scanopt.c
  src/scanopt.h
  src/sym.c
  src/tables.c
  src/tables.h
  src/tables_shared.c
  src/tables_shared.h
  src/tblcmp.c
  src/yylex.c
)

if(ENABLE_MEM_SHIM)
  list(APPEND sources lib/malloc.c lib/realloc.c)
endif()

# command line
add_executable(flex ${sources} src/main.c ${GENERATED_SOURCES})
add_library(libflex SHARED ${sources} ${GENERATED_SOURCES})
if(UNIX AND NOT ANDROID)
  target_link_libraries(flex PUBLIC m)
  target_link_libraries(libflex PUBLIC m)
  set(PKGCONFIG_LIBS "-lm -lc")
endif()
set_target_properties(libflex PROPERTIES
  OUTPUT_NAME fl
  EXPORT_NAME library
  SOVERSION "${FLEX_VERSION}"
)
add_executable(flex::flex ALIAS flex)
add_library(flex::library ALIAS libflex)

# Installation
install(TARGETS flex EXPORT flex_exports
  RUNTIME COMPONENT runtime)
install(TARGETS libflex EXPORT flex_exports
  ARCHIVE COMPONENT devel
  LIBRARY COMPONENT library NAMELINK_COMPONENT devel
  RUNTIME COMPONENT library
)
install(FILES "src/FLexLexer.h"
  DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}" COMPONENT devel)

# pkgconfig
set(prefix "${CMAKE_INSTALL_PREFIX}")
set(exec_prefix "\${prefix}")
file(RELATIVE_PATH libdir_relative "${CMAKE_INSTALL_PREFIX}" "${CMAKE_INSTALL_LIBDIR}")
set(libdir "\${exec_prefix}/${libdir_relative}")
configure_file(src/libfl.pc.in "${CMAKE_CURRENT_BINARY_DIR}/libfl.pc" @ONLY)
install(FILES "${CMAKE_CURRENT_BINARY_DIR}/libfl.pc"
  DESTINATION "${CMAKE_INSTALL_LIBDIR}/pkgconfig"
  COMPONENT devel)

# CMake
install(EXPORT flex_exports DESTINATION "${CMAKE_INSTALL_CMAKEDIR}"
  NAMESPACE flex:: FILE "FlexTargets.cmake")

include(CMakePackageConfigHelpers)

configure_package_config_file(cmake/FlexConfig.cmake.in FlexConfigVersion.cmake
    INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/flex/cmake
    PATH_VARS CMAKE_INSTALL_INCLUDEDIR)

write_basic_package_version_file(FlexConfigVersion.cmake
  VERSION ${PACKAGE_VERSION} COMPATIBILITY ExactVersion)

install(FILES "${CMAKE_CURRENT_BINARY_DIR}/FlexConfig.cmake" "${CMAKE_CURRENT_BINARY_DIR}/FlexConfigVersion.cmake"
  DESTINATION "${CMAKE_INSTALL_CMAKEDIR}"
  COMPONENT devel)

# uninstall
add_custom_target(uninstall "${CMAKE_COMMAND}" -P "${CMAKE_SOURCE_DIR}/cmake/make_uninstall.cmake")
