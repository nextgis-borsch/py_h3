################################################################################
# Project:  Lib Psycopg
# Purpose:  CMake build scripts
# Author:   Dmitry Baryshnikov, dmitry.baryshnikov@nexgis.com
################################################################################
# Copyright (C) 2017-2025, NextGIS <info@nextgis.com>
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
# THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING
# FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER
# DEALINGS IN THE SOFTWARE.
################################################################################

cmake_minimum_required(VERSION 3.15 FATAL_ERROR)

project(h3)

if(NOT DEFINED PACKAGE_VENDOR)
    set(PACKAGE_VENDOR NextGIS)
endif()

if(NOT DEFINED PACKAGE_BUGREPORT)
    set(PACKAGE_BUGREPORT info@nextgis.com)
endif()

# some init settings
set(CMAKE_COLOR_MAKEFILE ON)
# set path to additional CMake modules
set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake ${CMAKE_MODULE_PATH})

if(OSX_FRAMEWORK)
  set(INSTALL_LIB_DIR "Library" CACHE INTERNAL "Installation directory for libraries" FORCE)
else()
    set(INSTALL_LIB_DIR "." CACHE INTERNAL "Installation directory for libraries" FORCE)
endif()

find_package(Python3 COMPONENTS Interpreter Development)
if(NOT Python3_FOUND)
	message(FATAL_ERROR "Cannot find python 3")
endif()

set(PYTHON_EXECUTABLE ${Python3_EXECUTABLE})

file(DOWNLOAD "https://pypi.python.org/pypi/${PROJECT_NAME}/json" "${CMAKE_CURRENT_BINARY_DIR}/meta.json")

include(util)
get_compiler_version(COMPILER)

set(VERSION_MAX "3.7.7")
execute_process(COMMAND ${PYTHON_EXECUTABLE}
    ${CMAKE_SOURCE_DIR}/cmake/pypi_process.py ${PLUGIN} ${CMAKE_CURRENT_BINARY_DIR}/meta.json ${COMPILER} ${VERSION_MAX}
    OUTPUT_VARIABLE PACKAGE_DETAILS
    OUTPUT_STRIP_TRAILING_WHITESPACE
)

list(GET PACKAGE_DETAILS 0 DOWNLOAD_URL)
list(GET PACKAGE_DETAILS 1 VERSION)
list(GET PACKAGE_DETAILS 2 PROJECT_CPACK_FILENAME)

report_version(${PROJECT_NAME} ${VERSION})

# Download tarbal
set(OUT_DIR_NAME ${PROJECT_NAME}-${VERSION})
set(OUT_NAME "${OUT_DIR_NAME}.tar.gz")

if(NOT EXISTS "${CMAKE_CURRENT_BINARY_DIR}/${OUT_NAME}")
    status_message("Download ${DOWNLOAD_URL}")
    file(DOWNLOAD "${DOWNLOAD_URL}" "${CMAKE_CURRENT_BINARY_DIR}/${OUT_NAME}"
        SHOW_PROGRESS
    )
endif()

# Extract archive
if(NOT EXISTS ${CMAKE_CURRENT_BINARY_DIR}/${OUT_DIR_NAME})
    status_message("Extract ${CMAKE_CURRENT_BINARY_DIR}/${OUT_NAME}")
    execute_process(COMMAND ${CMAKE_COMMAND} -E tar xzf ${CMAKE_CURRENT_BINARY_DIR}/${OUT_NAME})
endif()


configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/cmake/CMakeLists.txt.cy.in
    ${OUT_DIR_NAME}/src/h3/_cy/CMakeLists.txt
    @ONLY
)
configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/cmake/CMakeLists.txt.in
    ${OUT_DIR_NAME}/CMakeLists.txt
    @ONLY
)


set(PY_H3_PROJECT py_h3)
include(ExternalProject)
ExternalProject_Add(
    ${PY_H3_PROJECT}
    PREFIX ${CMAKE_BINARY_DIR}
    SOURCE_DIR ${CMAKE_CURRENT_BINARY_DIR}/${OUT_DIR_NAME}
    BINARY_DIR ${CMAKE_BINARY_DIR}/${PY_H3_PROJECT}-build/
    INSTALL_DIR ${CMAKE_BINARY_DIR}/${PY_H3_PROJECT}-install/
    CMAKE_ARGS -DCMAKE_INSTALL_PREFIX=install
)

configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/cmake/_version.py.in
    ${OUT_DIR_NAME}/src/h3/_version.py
    @ONLY
)

file(REMOVE
    ${CMAKE_CURRENT_BINARY_DIR}/${OUT_DIR_NAME}/src/h3/api/basic_str/__init__.py
    ${CMAKE_CURRENT_BINARY_DIR}/${OUT_DIR_NAME}/src/h3/api/memview_int/__init__.py
    ${CMAKE_CURRENT_BINARY_DIR}/${OUT_DIR_NAME}/src/h3/api/numpy_int/__init__.py)

file(COPY_FILE
     ${CMAKE_CURRENT_BINARY_DIR}/${OUT_DIR_NAME}/src/h3/api/basic_int/__init__.py
     ${CMAKE_CURRENT_BINARY_DIR}/${OUT_DIR_NAME}/src/h3/api/basic_str/__init__.py)
file(COPY_FILE
     ${CMAKE_CURRENT_BINARY_DIR}/${OUT_DIR_NAME}/src/h3/api/basic_int/__init__.py
     ${CMAKE_CURRENT_BINARY_DIR}/${OUT_DIR_NAME}/src/h3/api/memview_int/__init__.py)
file(COPY_FILE
     ${CMAKE_CURRENT_BINARY_DIR}/${OUT_DIR_NAME}/src/h3/api/basic_int/__init__.py
     ${CMAKE_CURRENT_BINARY_DIR}/${OUT_DIR_NAME}/src/h3/api/numpy_int/__init__.py)

if(NOT SKIP_INSTALL_LIBRARIES AND NOT SKIP_INSTALL_ALL )
    install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/${OUT_DIR_NAME}/src/h3 DESTINATION ${INSTALL_LIB_DIR}/)
    install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/${OUT_DIR_NAME}/src/h3.egg-info DESTINATION ${INSTALL_LIB_DIR}/)
    install(DIRECTORY ${CMAKE_BINARY_DIR}/${PY_H3_PROJECT}-build/src/h3/_cy/Release/ DESTINATION ${INSTALL_LIB_DIR}/${PROJECT_NAME}/_cy/)
endif()

# Archiving ====================================================================
set(PACKAGE_NAME ${PROJECT_NAME})

set(CPACK_PACKAGE_NAME "${PACKAGE_NAME}")
set(CPACK_PACKAGE_VENDOR "${PACKAGE_VENDOR}")
set(CPACK_PACKAGE_VERSION "${VERSION}")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "${PACKAGE_NAME} Installation")
set(CPACK_PACKAGE_RELOCATABLE TRUE)
set(CPACK_ARCHIVE_COMPONENT_INSTALL ON)
set(CPACK_GENERATOR "ZIP")
set(CPACK_MONOLITHIC_INSTALL ON)
set(CPACK_STRIP_FILES TRUE)

# Get cpack zip archive name
set(CPACK_PACKAGE_FILE_NAME ${PROJECT_CPACK_FILENAME})

include(CPack)