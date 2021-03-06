##Copyright (c) 2011-2014 Thomas Paviot (tpaviot@gmail.com)
##
##This file is part of pythonOCC.
##
##pythonOCC is free software: you can redistribute it and/or modify
##it under the terms of the GNU General Public License as published by
##the Free Software Foundation, either version 3 of the License, or
##(at your option) any later version.
##
##pythonOCC is distributed in the hope that it will be useful,
##but WITHOUT ANY WARRANTY; without even the implied warranty of
##MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
##GNU General Public License for more details.
##
##You should have received a copy of the GNU General Public License
##along with pythonOCC.  If not, see <http://www.gnu.org/licenses/>.

project(PYTHONOCC)

# set pythonOCC version
set(PYTHONOCC_VERSION_MAJOR 0)
set(PYTHONOCC_VERSION_MINOR 16)
set(PYTHONOCC_VERSION_PATCH 1)
#  Empty for official releases, set to -dev, -rc1, etc for development releases
set(PYTHONOCC_VERSION_DEVEL -dev)

cmake_minimum_required(VERSION 2.6)

set(CMAKE_BUILD_TYPE "MinSizeRel" CACHE STRING "Build type")
message(STATUS " ${CMAKE_CXX_FLAGS}") 
set(BUILD_SHARED_LIBS ON)
include_directories(${CMAKE_CURRENT_SOURCE_DIR})

#############################
# Option_With_Default macro #
#############################
macro( option_with_default OPTION_NAME OPTION_STRING OPTION_DEFAULT )
if( NOT DEFINED ${OPTION_NAME} )
	set( ${OPTION_NAME} ${OPTION_DEFAULT} )
endif( NOT DEFINED ${OPTION_NAME} )
	OPTION( ${OPTION_NAME} "${OPTION_STRING}" ${${OPTION_NAME}} )
endmacro( option_with_default OPTION_NAME OPTION_STRING OPTION_DEFAULT )

##########
# Python #
##########
find_package(PythonInterp)
find_package(PythonLibs)
include_directories(${PYTHON_INCLUDE_PATH})

##############################################################
# 32/64 bit detection
###############################################################
if(${CMAKE_SIZEOF_VOID_P} MATCHES "8")
	# If in 64bit mode, we have to define the _OCC64 variable
	add_definitions(-D_OCC64)
	set(CMAKE_SWIG_FLAGS ${CMAKE_SWIG_FLAGS} -D_OCC64)
	set(BIT 64)
else(${CMAKE_SIZEOF_VOID_P} MATCHES "8")
	set(BIT 32)
endif(${CMAKE_SIZEOF_VOID_P} MATCHES "8")
message(STATUS "Build ${BIT}bit")

if ((NOT MSVC) OR (CMAKE_BUILD_TOOL STREQUAL "nmake"))
	if( NOT DEFINED PYTHONOCC_BUILD_TYPE )
		set( PYTHONOCC_BUILD_TYPE "MinSizeRel" CACHE STRING "Build type" ) # By default set release build
	endif( NOT DEFINED PYTHONOCC_BUILD_TYPE )
	set( CMAKE_BUILD_TYPE ${PYTHONOCC_BUILD_TYPE} CACHE INTERNAL "Build type,
		immutable" FORCE )
endif((NOT MSVC) OR (CMAKE_BUILD_TOOL STREQUAL "nmake"))

########
# SWIG #
########
find_package(SWIG)
include(${SWIG_USE_FILE})
set(SWIG_FILES_PATH src/SWIG_files/wrapper)
    
################################
# OCE include and lib location #
################################
find_package(OCE REQUIRED)
if(OCE_FOUND)
  message(STATUS "OpenCASCADE Community Edition found.")
  include_directories(${OCE_INCLUDE_DIRS})
  #set(OCE_LIBRARIES ${OCE_REQUIRED_LIBS})
else()
  set(OCE_include_PATH /usr/local/include/oce CACHE PATH "OCE include path" )
  set(OCE_LIB_PATH /usr/local/lib CACHE PATH "OCE lib path" )
  include_directories(${OCE_INCLUDE_PATH})
  link_directories(${OCE_LIB_PATH})
endif(OCE_FOUND)

##############################################
# Installation directory                     #
# by default, installed to site-packages/OCC #
##############################################
execute_process(COMMAND ${PYTHON_EXECUTABLE} -c "from __future__ import print_function; from distutils.sysconfig import get_python_lib; import os;print(get_python_lib())" OUTPUT_VARIABLE python_lib OUTPUT_STRIP_TRAILING_WHITESPACE )
set(PYTHONOCC_INSTALL_DIRECTORY ${python_lib}/OCC CACHE PATH "pythonOCC install directory" )

# List of OCE shared libraries to link with
# TODO: this should be build from the module list
set(OCE_MODEL_LIBRARIES TKernel TKMath TKAdvTools TKG2d TKG3d TKGeomBase TKBRep TKGeomAlgo
    TKTopAlgo TKPrim TKBO TKHLR TKMesh TKShHealing TKXMesh TKBool TKFillet TKFeat TKOffset)
set(OCE_VISUALIZATION_LIBRARIES TKService TKV3d TKMeshVS TKOpenGl)
set(OCE_DATAEXCHANGE_LIBRARIES TKSTL TKXSBase TKSTEPBase TKIGES TKSTEPAttr TKSTEP209
    TKSTEP TKVRML)

######################################################
# Configure Build settings according to the platform #
######################################################
if(UNIX)
  if(APPLE)
    ######### MacOSX ###########
	set(PLATFORM Darwin)
  else(APPLE)
    ######### Unix/Linux ###########
    set(PLATFORM Unix)
  endif(APPLE)
  set(CMAKE_SWIG_FLAGS -O -w302,401,314,509,512
        -DCSFDB -DHAVE_CONFIG_H -DOCC_CONVERT_SIGNALS)
  add_definitions(-DHAVE_CONFIG_H -DCSFDB -DOCC_CONVERT_SIGNALS)
else(UNIX)
  if(WIN32)
    ######### Windows ###########
    add_definitions(-DWNT -DWIN32 -D_WINDOWS -DCSFDB -DHAVE_CONFIG_H)
	# specific windows defines for python version >=2.6.0
	execute_process(COMMAND python -c "import sys;print (2,6)>=sys.version_info[:2]" OUTPUT_VARIABLE PYTHON_26_OR_HIGHER OUTPUT_STRIP_TRAILING_WHITESPACE)
	if(${PYTHON_26_OR_HIGHER} MATCHES "True")
	  add_definitions(-DHAVE_ACOSH -DHAVE_ASINH -DHAVE_ATANH)
	endif(${PYTHON_26_OR_HIGHER} MATCHES "True")
    set(CMAKE_SWIG_FLAGS -O -w302,401,314,509,512 -DCSFDB -DWIN32 -D_WINDOWS)
    set(PLATFORM win)
  else(WIN32)
    message(STATUS "Unknown platform")
  endif(WIN32)
endif(UNIX)
message(STATUS "Platform:" ${PLATFORM})

##################################
# Tells Swig to use py3k option  #
##################################
execute_process(COMMAND ${PYTHON_EXECUTABLE} -c "from __future__ import print_function; import sys; print(sys.version_info.major)" OUTPUT_VARIABLE python_version_major OUTPUT_STRIP_TRAILING_WHITESPACE )

if(${python_version_major} MATCHES "3")
  set(CMAKE_SWIG_FLAGS ${CMAKE_SWIG_FLAGS} -py3)
endif(${python_version_major} MATCHES "3")

################################################
# Define output path for generated libraries:  #
# platform/compiler-build_type-bits            #
# for instance:                                #
#     ./win32/bin/vc7.1-release-64             #
#     ./win32/bin/vc9-debug-32                 #
#     ./Unix/i386-debug-64                     #
################################################
if(WIN32)
  if(MSVC) # The compiler used is MSVC
    message(STATUS "Found MSVC compiler: ${MSVC} ${MSVC_VERSION}")
    set(LIBRARY_OUTPUT_PATH ${CMAKE_BINARY_DIR}/bin/)
  elseif (BORLAND) # The compiler used is BORLAND
    set(LIBRARY_OUTPUT_PATH ${CMAKE_BINARY_DIR}/win${BIT}/bin/${CMAKE_BUILD_TYPE})
  else()
    set(LIBRARY_OUTPUT_PATH win${BIT}/bin/${CMAKE_BUILD_TYPE})
  endif()
  set(EXECUTABLE_OUTPUT_PATH ${LIBRARY_OUTPUT_PATH})
else(WIN32)
  set(LIBRARY_OUTPUT_PATH ${PLATFORM}/${CMAKE_SYSTEM_PROCESSOR}-${CMAKE_BUILD_TYPE}-${BIT})
endif(WIN32)
set(CMAKE_SWIG_OUTDIR ${LIBRARY_OUTPUT_PATH})
message(STATUS "output_path: ${LIBRARY_OUTPUT_PATH}")

############################
# OpenCASCADE Modules list #
############################
include(${CMAKE_CURRENT_SOURCE_DIR}/cmake/OCE_Modules.cmake)
# add an option to choose toolkits to compile
option_with_default(PYTHONOCC_WRAP_VISU "Compile Visualisation" TRUE)
option_with_default(PYTHONOCC_WRAP_DATAEXCHANGE "Compile Data Exchange" TRUE)

###################
# Wrapper Modules #
###################

# MODEL Toolkits
file(MAKE_DIRECTORY ${CMAKE_BINARY_DIR}/${SWIG_FILES_PATH})
foreach(OCE_MODULE ${OCE_TOOLKIT_MODEL})
	set(FILE ${SWIG_FILES_PATH}/${OCE_MODULE}.i)
	set_source_files_properties(${FILE} PROPERTIES CPLUSPLUS ON)
  swig_add_module(${OCE_MODULE} python ${FILE})
  swig_link_libraries(${OCE_MODULE} ${PYTHON_LIBRARIES} ${OCE_MODEL_LIBRARIES})
endforeach(OCE_MODULE)

#################
# Visualisation #
#################
if(PYTHONOCC_WRAP_VISU)
  foreach(OCE_MODULE ${OCE_TOOLKIT_VISUALIZATION})
    set(FILE ${SWIG_FILES_PATH}/${OCE_MODULE}.i)
    set_source_files_properties(${FILE} PROPERTIES CPLUSPLUS ON)
    swig_add_module(${OCE_MODULE} python ${FILE})
    swig_link_libraries(${OCE_MODULE} ${PYTHON_LIBRARIES} ${OCE_MODEL_LIBRARIES} ${OCE_VISUALIZATION_LIBRARIES})
  endforeach(OCE_MODULE)
  # Build third part modules
  # TODO : the following line is strange but necessary
  execute_process(COMMAND ${CMAKE_COMMAND} -E make_directory src/Visualization)
  set_source_files_properties(${CMAKE_CURRENT_SOURCE_DIR}/src/Visualization/Visualization.i PROPERTIES CPLUSPLUS ON)
  include_directories(${CMAKE_CURRENT_SOURCE_DIR}/src/Visualization)
  set(VISUALIZATION_SOURCE_FILES
  	${CMAKE_CURRENT_SOURCE_DIR}/src/Visualization/Visualization.i
  	${CMAKE_CURRENT_SOURCE_DIR}/src/Visualization/Display3d.cpp
  	${CMAKE_CURRENT_SOURCE_DIR}/src/Visualization/Tesselator.cpp
  	)
  swig_add_module(Visualization python ${VISUALIZATION_SOURCE_FILES})
  if(APPLE)
    # on OSX, always add /System/Library/Frameworks/Cocoa.framework, even
    # if GLX is enabled
    swig_link_libraries(Visualization ${PYTHON_LIBRARIES} ${OCE_MODEL_LIBRARIES} ${OCE_VISUALIZATION_LIBRARIES} /System/Library/Frameworks/Cocoa.framework)
  else(APPLE)
    swig_link_libraries(Visualization ${PYTHON_LIBRARIES} ${OCE_MODEL_LIBRARIES} ${OCE_VISUALIZATION_LIBRARIES})
  endif(APPLE)
endif(PYTHONOCC_WRAP_VISU)

################
# DataExchange #
################
if(PYTHONOCC_WRAP_DATAEXCHANGE)
  foreach(OCE_MODULE ${OCE_TOOLKIT_DATAEXCHANGE})
    set(FILE ${SWIG_FILES_PATH}/${OCE_MODULE}.i)
    set_source_files_properties(${FILE} PROPERTIES CPLUSPLUS ON)
    swig_add_module(${OCE_MODULE} python ${FILE})
    swig_link_libraries(${OCE_MODULE} ${PYTHON_LIBRARIES} ${OCE_MODEL_LIBRARIES} ${OCE_DATAEXCHANGE_LIBRARIES})
  endforeach(OCE_MODULE)
endif(PYTHONOCC_WRAP_DATAEXCHANGE)

################
# Installation #
################
if(WIN32)
	set(EXTENSION "pyd")
else(WIN32)
	set(EXTENSION "so")
endif(WIN32)
set(BUILD_DIR ${CMAKE_BINARY_DIR}/${LIBRARY_OUTPUT_PATH})

# install pythonOCC modules
foreach(OCE_MODULE ${OCE_TOOLKIT_MODEL})
	install(FILES ${BUILD_DIR}/${OCE_MODULE}.py DESTINATION ${PYTHONOCC_INSTALL_DIRECTORY} )
	install(FILES ${BUILD_DIR}/_${OCE_MODULE}.${EXTENSION} DESTINATION ${PYTHONOCC_INSTALL_DIRECTORY} )
endforeach(OCE_MODULE)
if(PYTHONOCC_WRAP_VISU)
foreach(OCE_MODULE ${OCE_TOOLKIT_VISUALIZATION})
    install(FILES ${BUILD_DIR}/${OCE_MODULE}.py DESTINATION ${PYTHONOCC_INSTALL_DIRECTORY} )
    install(FILES ${BUILD_DIR}/_${OCE_MODULE}.${EXTENSION} DESTINATION ${PYTHONOCC_INSTALL_DIRECTORY} )
endforeach(OCE_MODULE)
endif(PYTHONOCC_WRAP_VISU)
if(PYTHONOCC_WRAP_DATAEXCHANGE)
foreach(OCE_MODULE ${OCE_TOOLKIT_DATAEXCHANGE})
    install(FILES ${BUILD_DIR}/${OCE_MODULE}.py DESTINATION ${PYTHONOCC_INSTALL_DIRECTORY} )
    install(FILES ${BUILD_DIR}/_${OCE_MODULE}.${EXTENSION} DESTINATION ${PYTHONOCC_INSTALL_DIRECTORY} )
endforeach(OCE_MODULE)
endif(PYTHONOCC_WRAP_DATAEXCHANGE)

# install third part modules
if(PYTHONOCC_WRAP_VISU)
install(FILES ${BUILD_DIR}/Visualization.py DESTINATION ${PYTHONOCC_INSTALL_DIRECTORY} )
install(FILES ${BUILD_DIR}/_Visualization.${EXTENSION} DESTINATION ${PYTHONOCC_INSTALL_DIRECTORY} )
endif(PYTHONOCC_WRAP_VISU)

# install GC
install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/src/GarbageCollector.py DESTINATION ${PYTHONOCC_INSTALL_DIRECTORY} )
# install addons
install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/src/addons/Display DESTINATION ${PYTHONOCC_INSTALL_DIRECTORY} )
# install LICENSE file
install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/LICENSE DESTINATION ${PYTHONOCC_INSTALL_DIRECTORY} )
# install AUTHORS file
install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/AUTHORS DESTINATION ${PYTHONOCC_INSTALL_DIRECTORY} )
# install __init__.py file
install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/cmake/__init__.py DESTINATION ${PYTHONOCC_INSTALL_DIRECTORY} )

#######################
# pythonOCC Packaging #
# $ make package      #
#######################
if(UNIX)
  include(InstallRequiredSystemLibraries)
endif(UNIX)
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "pythonOCC")
set(CPACK_PACKAGE_VENDOR "Thomas Paviot (tpaviot@gmail.com)")
set(CPACK_SET_DESTDIR "ON")
set(CPACK_PACKAGE_DESCRIPTION "A 3D CAD/CAM python package")
set(CPACK_PACKAGE_VERSION "0.16.0")
set(CPACK_PACKAGE_VERSION_MAJOR ${PYTHONOCC_VERSION_MAJOR})
set(CPACK_PACKAGE_VERSION_MINOR ${PYTHONOCC_VERSION_MINOR})
set(CPACK_PACKAGE_VERSION_PATCH ${PYTHONOCC_VERSION_PATCH})
set(CPACK_PACKAGE_NAME PYTHONOCC)
set(CPACK_RESOURCE_FILE_LICENSE ${CMAKE_CURRENT_SOURCE_DIR}/LICENSE)
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "A 3D CAD/CAM python package")
set(CPACK_PACKAGE_CONTACT "pythonocc googlegroup <pythonocc@googlegroups.com>")

##########
# Debian #
##########
set(CPACK_GENERATOR "DEB")
set(CPACK_DEBIAN_PACKAGE_NAME "pythonocc")
set(CPACK_DEBIAN_PACKAGE_SECTION "devel")
set(CPACK_DEBIAN_PACKAGE_PRIORITY "optional")
set(CPACK_DEBIAN_PACKAGE_MAINTAINER "Thomas Paviot (tpaviot@gmail.com)")
set(CPACK_DEBIAN_PACKAGE_HOMEPAGE "http://www.pythonocc.org")
include(CPack)
