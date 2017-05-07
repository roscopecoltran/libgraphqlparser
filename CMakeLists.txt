cmake_minimum_required(VERSION 3.0)

string(COMPARE NOTEQUAL "$ENV{TRAVIS_TAG}" "" travis_deploy)
string(COMPARE EQUAL "$ENV{APPVEYOR_REPO_TAG}" "true" appveyor_deploy)

if(travis_deploy)
  set(version "$ENV{TRAVIS_TAG}")
elseif(appveyor_deploy)
  set(version "$ENV{APPVEYOR_REPO_TAG_NAME}")
else()
  set(version "v0.6.0")
endif()

string(REGEX REPLACE "^v" "" version "${version}")

include("cmake/HunterGate.cmake")
HunterGate(
  URL "https://github.com/ruslo/hunter/archive/v0.18.18.tar.gz"
  SHA1 "ae1f72b9e8779a58f763179c77f609c0587d26cd"
  )

PROJECT(libgraphqlparser VERSION ${version} LANGUAGES C CXX)

list(APPEND CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake")

OPTION(test "Build tests." OFF)

set_property(GLOBAL PROPERTY USE_FOLDERS ON)

set(is_android ${ANDROID}) # syntax compatibility
string(COMPARE EQUAL "${CMAKE_OSX_SYSROOT}" "iphoneos" is_ios)
string(COMPARE EQUAL "${CMAKE_SYSTEM_NAME}" "Linux" is_linux)
string(COMPARE EQUAL "${CMAKE_GENERATOR}" "Xcode" is_xcode)

# See: cmake/Hunter/config.cmake
# First call to 'hunter_add_package' will contains read of Hunter/config.cmake
# Next variables will be used:
#   * is_ios
#   * is_linux
hunter_add_package(Sugar)
include("${SUGAR_ROOT}/cmake/Sugar")
include(sugar_include)

if(is_xcode)
  option(${PROJECT_NAME}_DISABLE_DSYM "Disable generation of dSYM information" ON)
  if(${PROJECT_NAME}_DISABLE_DSYM)
    set(CMAKE_XCODE_ATTRIBUTE_DEBUG_INFORMATION_FORMAT "dwarf")
  endif()
  set(CMAKE_CONFIGURATION_TYPES "Debug;Release;MinSizeRel;RelWithDebInfo" CACHE STRING "Configuration types")
  set(CMAKE_XCODE_ATTRIBUTE_ENABLE_BITCODE NO)
endif()

INCLUDE(version)

SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=gnu++11")

FIND_PACKAGE(PythonInterp 2 REQUIRED)
IF (NOT PYTHON_VERSION_MAJOR EQUAL 2)
  MESSAGE(FATAL_ERROR "Python 2 is required.")
ENDIF()

FIND_PROGRAM(CTYPESGEN_FOUND ctypesgen.py)

FIND_PACKAGE(BISON 3)
FIND_PACKAGE(FLEX)
IF (BISON_FOUND)
  BISON_TARGET(graphqlparser parser.ypp ${CMAKE_CURRENT_SOURCE_DIR}/parser.tab.cpp)
ENDIF()

IF(FLEX_FOUND)
  FLEX_TARGET(GraphQLScanner lexer.lpp ${CMAKE_CURRENT_SOURCE_DIR}/lexer.cpp COMPILE_FLAGS "--header-file=lexer.h")
  IF (BISON_FOUND)
    ADD_FLEX_BISON_DEPENDENCY(GraphQLScanner graphqlparser)
  ENDIF()
ENDIF()

FILE(MAKE_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/c)
INCLUDE_DIRECTORIES(${CMAKE_CURRENT_SOURCE_DIR})
INCLUDE_DIRECTORIES(${CMAKE_CURRENT_BINARY_DIR})

set(graphqlparser_src
  JsonVisitor.cpp
  ${CMAKE_CURRENT_BINARY_DIR}/Ast.h
  ${CMAKE_CURRENT_BINARY_DIR}/Ast.cpp
  ${CMAKE_CURRENT_BINARY_DIR}/AstVisitor.h
  ${CMAKE_CURRENT_BINARY_DIR}/c/GraphQLAst.h
  ${CMAKE_CURRENT_BINARY_DIR}/c/GraphQLAst.cpp
  ${CMAKE_CURRENT_BINARY_DIR}/c/GraphQLAstForEachConcreteType.h
  ${CMAKE_CURRENT_BINARY_DIR}/JsonVisitor.h.inc
  ${CMAKE_CURRENT_BINARY_DIR}/JsonVisitor.cpp.inc
  c/GraphQLAstNode.cpp
  c/GraphQLAstToJSON.cpp
  c/GraphQLAstVisitor.h
  c/GraphQLAstVisitor.cpp
  c/GraphQLParser.cpp
  parser.tab.cpp
  parser.tab.hpp
  lexer.cpp
  lexer.h
  GraphQLParser.cpp)


ADD_LIBRARY(graphqlparser STATIC ${graphqlparser_src})
ADD_LIBRARY(graphqlparser_s SHARED ${graphqlparser_src})
set_target_properties(graphqlparser_s PROPERTIES OUTPUT_NAME graphqlparser)

# Enable this and remove CMAKE_CXX_FLAGS fiddle above when we are able
# to upgrade to CMake 2.8.12. Blocker seems to be Travis CI being on
# Ubuntu Precise; Trusty has 2.8.12.
# TARGET_COMPILE_OPTIONS(graphqlparser PUBLIC -std=gnu++11)

ADD_EXECUTABLE(dump_json_ast dump_json_ast.cpp)
TARGET_LINK_LIBRARIES(dump_json_ast graphqlparser)

FUNCTION(GENERATE_AST_FILE FILE_TYPE FILE_RELATIVE_PATH)
  ADD_CUSTOM_COMMAND(
    OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/${FILE_RELATIVE_PATH}
    COMMAND ${PYTHON_EXECUTABLE} ${CMAKE_CURRENT_SOURCE_DIR}/ast/ast.py ${FILE_TYPE} ${CMAKE_CURRENT_SOURCE_DIR}/ast/ast.ast > ${CMAKE_CURRENT_BINARY_DIR}/${FILE_RELATIVE_PATH}
    DEPENDS ${CMAKE_CURRENT_SOURCE_DIR}/ast/ast.ast ${CMAKE_CURRENT_SOURCE_DIR}/ast/ast.py ${CMAKE_CURRENT_SOURCE_DIR}/ast/${FILE_TYPE}.py)
ENDFUNCTION(GENERATE_AST_FILE)

GENERATE_AST_FILE(cxx Ast.h)

GENERATE_AST_FILE(cxx_visitor AstVisitor.h)

GENERATE_AST_FILE(cxx_impl Ast.cpp)

GENERATE_AST_FILE(c c/GraphQLAst.h)

GENERATE_AST_FILE(c_impl c/GraphQLAst.cpp)

GENERATE_AST_FILE(c_visitor_impl c/GraphQLAstForEachConcreteType.h)

GENERATE_AST_FILE(cxx_json_visitor_header JsonVisitor.h.inc)

GENERATE_AST_FILE(cxx_json_visitor_impl JsonVisitor.cpp.inc)

ADD_SUBDIRECTORY(python)

### Install ###
set(generated_dir "${CMAKE_CURRENT_BINARY_DIR}/generated")

set(config_install_dir "lib/cmake/${PROJECT_NAME}")
set(include_install_dir "include")

set(version_config "${generated_dir}/${PROJECT_NAME}ConfigVersion.cmake")
set(project_config "${generated_dir}/${PROJECT_NAME}Config.cmake")
set(targets_export_name "${PROJECT_NAME}Targets")
set(namespace "${PROJECT_NAME}::")

include(CMakePackageConfigHelpers)
write_basic_package_version_file(
    "${version_config}" COMPATIBILITY SameMajorVersion
)

# Note: use 'targets_export_name'
configure_package_config_file(
    "${PROJECT_SOURCE_DIR}/cmake/Config.cmake.in"
    "${project_config}"
     INSTALL_DESTINATION "${config_install_dir}"
)

install(
    TARGETS graphqlparser graphqlparser_s dump_json_ast
    EXPORT "${targets_export_name}"
    LIBRARY DESTINATION "lib"
    ARCHIVE DESTINATION "lib"
    RUNTIME DESTINATION "bin"
    INCLUDES DESTINATION "${include_install_dir}"
    )

install(
  FILES "${project_config}" "${version_config}"
  DESTINATION "${config_install_dir}"
  )

install(
  EXPORT "${targets_export_name}"
  NAMESPACE "${namespace}"
  DESTINATION "${config_install_dir}"
  )

install(
  DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/c
  DESTINATION "${include_install_dir}/${PROJECT_NAME}" 
  FILES_MATCHING PATTERN "*.h"
  PATTERN "build" EXCLUDE
  ) 

#INSTALL(DIRECTORY c ${CMAKE_CURRENT_BINARY_DIR}/c DESTINATION include/graphqlparser
#  FILES_MATCHING PATTERN "*.h"
#  PATTERN "build" EXCLUDE)

INSTALL(FILES
  ${CMAKE_CURRENT_BINARY_DIR}/Ast.h
  AstNode.h
  ${CMAKE_CURRENT_BINARY_DIR}/AstVisitor.h
  GraphQLParser.h
  JsonVisitor.h
  lexer.h
  location.hh
  parser.tab.hpp
  position.hh
  stack.hh
  syntaxdefs.h
  DESTINATION "${include_install_dir}/${PROJECT_NAME}"
)

#INSTALL(TARGETS graphqlparser
#  LIBRARY DESTINATION lib)

if (UNIX)
  # generate pkgconfig file
  include(FindPkgConfig QUIET)
  if(PKG_CONFIG_FOUND)
    # generate .pc and install
    configure_file("libgraphqlparser.pc.in" "libgraphqlparser.pc" @ONLY)
    install(FILES       "${CMAKE_CURRENT_BINARY_DIR}/libgraphqlparser.pc"
            DESTINATION "${CMAKE_INSTALL_PREFIX}/lib/pkgconfig")
  endif()
endif()

IF (test)

  ADD_SUBDIRECTORY(test)

  if(UNIX)
    # setup valgrind
    ADD_CUSTOM_TARGET(memcheck
      valgrind --leak-check=full --suppressions=./test/valgrind.supp --dsymutil=yes --error-exitcode=1 ./test/runTests  >/dev/null
    )
  endif()

ENDIF()
