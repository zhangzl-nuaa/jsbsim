project(jsbsim CXX C)

# settings
cmake_minimum_required(VERSION 2.8)
set(PROJECT_VERSION_MAJOR "1")
set(PROJECT_VERSION_MINOR "0")
set(PROJECT_VERSION_PATCH "0.108")
set(PROJECT_VERSION "${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}.${PROJECT_VERSION_PATCH}")
set(PROJECT_CONTACT_EMAIL james.goppert@gmail.com)
set(PROJECT_CONTACT_VENDOR JSBSim)
set(LIBRARY_VERSION ${PROJECT_VERSION})
set(LIBRARY_SOVERSION "0.0.0")

# third party
set(ARKOSG_VERSION "13d53e1b793ca8b82893980a19f11a70c4b1d450")
set(ARKCOMM_VERSION "0c224b528ede947d410d9a56e2b7a47d6ae86048")

# options
option(WITH_BUILD_DEPS "Build dependencies." OFF)
option(WITH_BUILD_SHARED "Build shared library." OFF)
option(WITH_PREFER_STATIC "Build preferring static linking." ON)
option(WITH_GUI "Build with GUI." OFF)
option(WITH_ARKOSG "Build arkosg graphics." OFF)
option(WITH_ARKCOMM "Build arkcomm communications." OFF)
option(WITH_BUNDLE "Attempt to package external shared library dependencies." OFF) # not working

# variables
set(ROOT_THREAD TRUE CACHE INTERNAL "Is this the top level of the recursion?")
set(INSTALL_DATA_DIR "${CMAKE_INSTALL_PREFIX}/share/${PROJECT_NAME}")
if (MINGW)
    set(CMAKE_INSTALL_PREFIX "/")
    set(INSTALL_DATA_DIR "C:/Program Files/${PROJECT_NAME} ${PROJECT_VERSION}/share/${PROJECT_NAME}")
endif()

# modules
list(APPEND CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake ${CMAKE_SOURCE_DIR}/cmake/arkcmake)
include(CheckIncludeFiles)
include(CheckFunctionExists)
include(CheckSymbolExists)
include(CheckLibraryExists)
#include(CheckTypeSize)
#include(CheckPrototypeExists)
#include(CheckCXXSourceCompiles)
#include(CheckCSourceCompiles)
include(ExternalProject)
include(FindARKOSG)
include(FindARKCOMM)

# spawn new cmake to build deps
if (WITH_BUILD_DEPS AND ROOT_THREAD)
    execute_process(COMMAND ${CMAKE_COMMAND} "${CMAKE_SOURCE_DIR}"
        "-DCMAKE_TOOLCHAIN_FILE=${CMAKE_TOOLCHAIN_FILE}"
        "-DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}"
        "-DEP_BASE_DIR=${EP_BASE_DIR}"
        "-DWITH_BUILD_DEPS=${WITH_BUILD_DEPS}"
        "-DWITH_BUILD_SHARED=${WITH_BUILD_SHARED}"
        "-DWITH_PREFER_STATIC=${WITH_PREFER_STATIC}"
        "-DWITH_GUI=${WITH_GUI}"
        "-DWITH_ARKOSG=${WITH_ARKOSG}"
        "-DWITH_ARKCOMM=${WITH_ARKCOMM}"
        "-DWITH_BUNDLE=${WITH_BUNDLE}"
        "-DROOT_THREAD=FALSE"
        RESULT_VARIABLE ERROR)
    if (ERROR)
        message(FATAL_ERROR "error, recursing loop returned error code: ${ERROR}")
    endif()
    message("** Making dependencies")
    execute_process(COMMAND ${CMAKE_MAKE_PROGRAM} "-j4" "-f${CMAKE_BINARY_DIR}/Makefile")
    message("** Configuring ${PROJECT_NAME}")
endif()

# external projects find path
if(NOT EP_BASE_DIR)
    set(EP_BASE_DIR "${CMAKE_BINARY_DIR}/CMakeExternals")
endif()
list(APPEND CMAKE_FIND_ROOT_PATH ${EP_BASE_DIR})

# prefer static packages if building static library
message("** Finding libraries")
if (WITH_PREFER_STATIC)
    set(BOOST_USE_STATIC_LIBS ON)
    # prefer static libs
    if(WIN32)
        set(CMAKE_FIND_LIBRARY_SUFFIXES .lib .a ${CMAKE_FIND_LIBRARY_SUFFIXES})
    else()
        set(CMAKE_FIND_LIBRARY_SUFFIXES .a ${CMAKE_FIND_LIBRARY_SUFFIXES})
    endif()
endif()

# find libraries with cmake modules
set(REQUIRED_IF_ROOT_THREAD)
if (ROOT_THREAD)
    set(REQUIRED_IF_ROOT_THREAD "REQUIRED")
endif()
if (WITH_GUI AND WITH_ARKOSG)
    if (MSVC)
        set(Boost_USE_MULITHREADED ON)
        find_package(Boost 1.40.0 COMPONENTS thread system REQUIRED)
    elseif(MINGW)
        find_package(Boost 1.40.0 COMPONENTS thread_win32-mt system-mt REQUIRED)
    else()
        find_package(Boost 1.40.0 COMPONENTS thread-mt system-mt REQUIRED)
    endif()
    find_package(OpenGL REQUIRED)
    find_package(Qt4 COMPONENTS QtOpenGL QtGui QtCore REQUIRED)
    find_package(OpenSceneGraph 2.8.0 ${REQUIRED_IF_ROOT_THREAD} COMPONENTS osgViewer osgUtil osgDB osgGA osgManipulator)

    # find osg plugins
    if (OPENSCENEGRAPH_FOUND)
        get_filename_component(OSG_LIBRARY_ROOT "${OSG_LIBRARY}" PATH)
        find_library(OSG_AC
            NAMES osgdb_ac
            PATHS ${OSG_LIBRARY_ROOT}/osgPlugins-${OPENSCENEGRAPH_VERSION}
            )
        find_library(OSG_RGB
            NAMES osgdb_rgb
            PATHS ${OSG_LIBRARY_ROOT}/osgPlugins-${OPENSCENEGRAPH_VERSION}
            )
        set(OSG_PLUGINS_PROCESS_LIBS OSG_AC OSG_RGB)
        set(OSG_PLUGINS_PROCESS_INCLUDES)
        libfind_process(OSG_PLUGINS)
    endif()

    find_package(ARKOSG ${REQUIRED_IF_ROOT_THREAD})
elseif(WITH_GUI)
    find_package(Qt4 COMPONENTS QtGui QtCore REQUIRED)
endif()
if(WITH_ARKCOMM)
    find_package(ARKCOMM ${REQUIRED_IF_ROOT_THREAD})
endif()

# build dependencies
if (WITH_BUILD_DEPS AND (NOT ROOT_THREAD) )
    message("** Configuring dependencies")

    # add external projects

    set(CMAKE_EP_DEFAULT_ARGS
        -DEP_BASE_DIR=${EP_BASE_DIR}
        -DCMAKE_TOOLCHAIN_FILE=${CMAKE_TOOLCHAIN_FILE}
        -DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}
        )

    if (WITH_ARKCOMM  AND (NOT ARKCOMM_FOUND))
        ExternalProject_Add(arkcomm
            GIT_REPOSITORY "git://github.com/arktools/arkcomm.git"
            GIT_TAG ${TAG}
            UPDATE_COMMAND ""
            INSTALL_DIR ${EP_BASE_DIR}${CMAKE_INSTALL_PREFIX}
            CMAKE_ARGS ${CMAKE_EP_DEFAULT_ARGS}
            INSTALL_COMMAND ${CMAKE_MAKE_PROGRAM} DESTDIR=${EP_BASE_DIR} install
           )
    endif()

    if (NOT QT_FOUND)
        if (APPLE)
            set(QT_FILE qt-mac-opensource-4.8.0.dmg)
            set(QT_URL http://get.qt.nokia.com/qt/source)
            ExternalProject_Add(osg
                DOWNLOAD_COMMAND curl -C - --create-dirs -o ${EP_BASE_DIR}/download/${QT_FILE} ${QT_URL}/${QT_FILE}
                DOWNLOAD_DIR ${EP_BASE_DIR}/download
                INSTALL_DIR ${EP_BASE_DIR}${CMAKE_INSTALL_PREFIX}
                INSTALL_COMMAND hdiutil mount ${EP_BASE_DIR}/download/${QT_FILE}
               )
        endif()
    endif()

    if (NOT OPENSCENEGRAPH_FOUND)
        if (APPLE)
            set(OSG_FILE OpenSceneGraph-2.8.5-osx-x86-release-12493)
            set(OSG_URL  http://openscenegraph.alphapixel.com/sites/default/files/osgdownload/2.8.5)
            message(STATUS "mv /tmp/${OSG_FILE}/* ${EP_BASE_DIR}${CMAKE_INSTALL_PREFIX}")
            ExternalProject_Add(osg
                DOWNLOAD_COMMAND curl -C - --create-dirs -o ${EP_BASE_DIR}/download/${OSG_FILE}.7z ${OSG_URL}/${OSG_FILE}.7z
                DOWNLOAD_DIR ${EP_BASE_DIR}/download
                CONFIGURE_COMMAND ""
                UPDATE_COMMAND ""
                BUILD_COMMAND ""
                INSTALL_DIR ${EP_BASE_DIR}${CMAKE_INSTALL_PREFIX}
                INSTALL_COMMAND  7z x ${EP_BASE_DIR}/download/${OSG_FILE}.7z -o${EP_BASE_DIR}/download
                    && mkdir -p ${EP_BASE_DIR}${CMAKE_INSTALL_PREFIX}/lib
                    && mkdir -p ${EP_BASE_DIR}${CMAKE_INSTALL_PREFIX}/include
                    && rsync -r ${EP_BASE_DIR}/download/${OSG_FILE}/lib/ ${EP_BASE_DIR}${CMAKE_INSTALL_PREFIX}/lib 
                    && rsync -r ${EP_BASE_DIR}/download/${OSG_FILE}/include/ ${EP_BASE_DIR}${CMAKE_INSTALL_PREFIX}/include
               )
        endif()
    endif()

    if (WITH_ARKOSG AND WITH_GUI AND (NOT ARKOSG_FOUND))
        ExternalProject_Add(arkosg
            GIT_REPOSITORY "git://github.com/arktools/arkosg.git"
            GIT_TAG ${TAG}
            UPDATE_COMMAND ""
            INSTALL_DIR ${EP_BASE_DIR}${CMAKE_INSTALL_PREFIX}
            CMAKE_ARGS ${CMAKE_EP_DEFAULT_ARGS}
            INSTALL_COMMAND ${CMAKE_MAKE_PROGRAM} DESTDIR=${EP_BASE_DIR} install
           )
       if (NOT OPENSCENEGRAPH_FOUND)
           add_dependencies(arkosg osg)
       endif()
    endif()

    # terminate non root cmake thread
    return()
endif()

# configure
check_include_files(expat_config.h HAVE_EXPAT_CONFIG_H)
check_include_files(sys_types.h HAVE_SYS_TYPES_H)
check_include_files(sys_stat.h HAVE_SYS_STAT_H)
check_include_files(stdlib.h HAVE_STDLIB_H)
check_include_files(string.h HAVE_STRING_H)
check_include_files(inttypes.h HAVE_INTTYPES_H)
check_include_files(stdint.h HAVE_STDINT_H)
check_include_files(unistd.h HAVE_UNISTD_H)
check_function_exists(memcopy HAVE_MEMCOPY)
check_function_exists(memmove HAVE_MEMMOVE)
check_function_exists(bcopy HAVE_BCOPY)
#check_symbol_exists(LC_MESSAGES "locale.h" HAVE_LC_MESSAGES) 
#check_library_exists(arkosg  attachNode "" HAVE_ARKOSG)

# config files
configure_file(config.h.in config.h)
configure_file(gui/gui_config.h.in gui/gui_config.h)

# build settings
if (HAVE_EXPAT_CONFIG_H OR MSVC)
    add_definitions(-DHAVE_EXPAT_CONFIG_H)
endif()
if (HAVE_MEMMOVE)
    add_definitions(-DHAVE_MEMMOVE)
endif()
if (HAVE_BCOPY)
    add_definitions(-DHAVE_BCOPY)
endif()
if (HAVE_EXPAT_CONFIG_H)
    add_definitions(-DHAVE_EXPAT_CONFIG_H)
endif()
if (MSVC)
    add_definitions(-D_USE_MATH_DEFINES)
elseif(MINGW)
    add_definitions(-D_WIN32_WINNT=0x0501)
elseif(APPLE)
elseif(UNIX)
    add_definitions(-DUSE_X11)
else()
    message(ERROR "unknown operating system")
endif()
include_directories(src
   src/simgear/xml)

# headers
set(MATH_HDRS
    src/math/FGColumnVector3.h
    src/math/FGCondition.h
    src/math/FGFunction.h
    src/math/FGLocation.h
    src/math/FGMatrix33.h
    src/math/FGModelFunctions.h
    src/math/FGNelderMead.h
    src/math/FGParameter.h
    src/math/FGPropertyValue.h
    src/math/FGQuaternion.h
    src/math/FGRealValue.h
    src/math/FGRungeKutta.h
    src/math/FGStateSpace.h
    src/math/FGTable.h
    src/math/LagrangeMultiplier.h
    )
install(FILES ${MATH_HDRS} DESTINATION include/jsbsim/math COMPONENT Dev)

set(SIMGEAR_HDRS
    src/simgear/compiler.h
    )
install(FILES ${SIMGEAR_HDRS} DESTINATION include/jsbsim/simgear COMPONENT Dev)

set(SIMGEAR_PROPS_HDRS
    src/simgear/props/props.hxx
    )
install(FILES ${SIMGEAR_PROPS_HDRS} DESTINATION include/jsbsim/simgear/props COMPONENT Dev)

set(SIMGEAR_MAGVAR_HDRS
    src/simgear/magvar/coremag.hxx
    )
install(FILES ${SIMGEAR_MAGVAR_HDRS} DESTINATION include/jsbsim/simgear/magvar COMPONENT Dev)

set(SIMGEAR_MISC_HDRS
    src/simgear/misc/stdint.hxx
    )
install(FILES ${SIMGEAR_MISC_HDRS} DESTINATION include/jsbsim/simgear/misc COMPONENT Dev)

set(SIMGEAR_STRUCTURE_HDRS
    src/simgear/structure/SGReferenced.hxx
    src/simgear/structure/SGSharedPtr.hxx

    )
install(FILES ${SIMGEAR_STRUCTURE_HDRS} DESTINATION include/jsbsim/simgear/structure COMPONENT Dev)


set(SIMGEAR_XML_HDRS
    src/simgear/xml/utf8tab.h
    src/simgear/xml/internal.h
    src/simgear/xml/ascii.h
    src/simgear/xml/expat.h
    src/simgear/xml/nametab.h
    src/simgear/xml/winconfig.h
    #src/simgear/xml/xmltok_impl.h
    src/simgear/xml/easyxml.hxx
    src/simgear/xml/macconfig.h
    src/simgear/xml/expat_config.h
    src/simgear/xml/iasciitab.h
    src/simgear/xml/xmlrole.h
    src/simgear/xml/expat_external.h
    src/simgear/xml/latin1tab.h
    src/simgear/xml/asciitab.h
    src/simgear/xml/xmltok.h
    )
install(FILES ${SIMGEAR_XML_HDRS} DESTINATION include/jsbsim/simgear/xml COMPONENT Dev)

set(INPUT_OUTPUT_HDRS
    src/input_output/FGXMLElement.h
    src/input_output/FGXMLParse.h
    src/input_output/FGfdmSocket.h
    src/input_output/string_utilities.h
    src/input_output/FGXMLFileRead.h
    src/input_output/net_fdm.hxx
    src/input_output/FGScript.h
    src/input_output/FGGroundCallback.h
    src/input_output/FGPropertyManager.h
    )
install(FILES ${INPUT_OUTPUT_HDRS} DESTINATION include/jsbsim/input_output COMPONENT Dev)

set(MODELS_HDRS
    src/models/FGAccelerations.h
    src/models/FGMassBalance.h
    src/models/FGFCS.h
    src/models/FGLGear.h
    src/models/FGInput.h
    src/models/FGAerodynamics.h
    src/models/FGAuxiliary.h
    src/models/FGBuoyantForces.h
    src/models/FGInertial.h
    src/models/FGPropulsion.h
    src/models/FGOutput.h
    src/models/FGExternalReactions.h
    src/models/FGAtmosphere.h
    src/models/atmosphere/FGMSIS.h
    src/models/atmosphere/FGMars.h
    src/models/FGAircraft.h
    src/models/FGExternalForce.h
    src/models/FGModel.h
    src/models/FGGasCell.h
    src/models/FGPropagate.h
    src/models/FGGroundReactions.h
    )
install(FILES ${MODELS_HDRS} DESTINATION include/jsbsim/models COMPONENT Dev)

set(MODELS_ATMOSPHERE_HDRS
    src/models/atmosphere/FGMSIS.h
    src/models/atmosphere/FGMars.h
    src/models/atmosphere/FGWinds.h
    src/models/atmosphere/FGStandardAtmosphere.h
    )
install(FILES ${MODELS_ATMOSPHERE_HDRS} DESTINATION include/jsbsim/models/atmosphere COMPONENT Dev)

set(MODELS_FLIGHT_CONTROL_HDRS
    src/models/flight_control/FGAccelerometer.h
    src/models/flight_control/FGFCSFunction.h
    src/models/flight_control/FGDeadBand.h
    src/models/flight_control/FGSensorOrientation.h
    src/models/flight_control/FGGyro.h
    src/models/flight_control/FGGain.h
    src/models/flight_control/FGSwitch.h
    src/models/flight_control/FGFCSComponent.h
    src/models/flight_control/FGSummer.h
    src/models/flight_control/FGPID.h
    src/models/flight_control/FGActuator.h
    src/models/flight_control/FGSensor.h
    src/models/flight_control/FGFilter.h
    src/models/flight_control/FGMagnetometer.h
    src/models/flight_control/FGKinemat.h
    )
install(FILES ${MODELS_FLIGHT_CONTROL_HDRS} DESTINATION include/jsbsim/models/flight_control COMPONENT Dev)

set(MODELS_PROPULSION_HDRS
    src/models/propulsion/FGElectric.h
    src/models/propulsion/FGTurbine.h
    src/models/propulsion/FGThruster.h
    src/models/propulsion/FGRocket.h
    src/models/propulsion/FGNozzle.h
    src/models/propulsion/FGTurboProp.h
    src/models/propulsion/FGTransmission.h
    src/models/propulsion/FGForce.h
    src/models/propulsion/FGRotor.h
    src/models/propulsion/FGPiston.h
    src/models/propulsion/FGEngine.h
    src/models/propulsion/FGTank.h
    src/models/propulsion/FGPropeller.h
    )
install(FILES ${MODELS_PROPULSION_HDRS} DESTINATION include/jsbsim/models/propulsion COMPONENT Dev)

set(INITIALIZATION_HDRS
    src/initialization/FGInitialCondition.h
    src/initialization/FGSimplexTrim.h
    #src/initialization/FGTrimAnalysisControl.h
    #src/initialization/FGTrimAnalysis.h
    src/initialization/FGTrimAxis.h
    src/initialization/FGTrim.h
    src/initialization/FGTrimmer.h
    )
install(FILES ${INITIALIZATION_HDRS} DESTINATION include/jsbsim/initialization COMPONENT Dev)

set(TOP_HDRS
    src/FGFDMExec.h
    src/FGJSBBase.h
    )
install(FILES ${TOP_HDRS} DESTINATION include/jsbsim COMPONENT Dev)

# install data files
install(DIRECTORY "aircraft" DESTINATION share/${PROJECT_NAME} COMPONENT Runtime PATTERN "*.git*" EXCLUDE)
install(DIRECTORY "data_output" DESTINATION share/${PROJECT_NAME} COMPONENT Runtime PATTERN "*.git*" EXCLUDE)
install(DIRECTORY "data_plot" DESTINATION share/${PROJECT_NAME} COMPONENT Runtime PATTERN "*.git*" EXCLUDE)
install(DIRECTORY "engine" DESTINATION share/${PROJECT_NAME} COMPONENT Runtime PATTERN "*.git*" EXCLUDE)
install(DIRECTORY "scripts" DESTINATION share/${PROJECT_NAME} COMPONENT Runtime PATTERN "*.git*" EXCLUDE)
install(DIRECTORY "engine" DESTINATION share/${PROJECT_NAME} COMPONENT Runtime PATTERN "*.git*" EXCLUDE)
install(DIRECTORY "systems" DESTINATION share/${PROJECT_NAME} COMPONENT Runtime PATTERN "*.git*" EXCLUDE)

# sources
set(JSBSIM_SRCS
    src/input_output/FGfdmSocket.cpp
    src/input_output/FGXMLParse.cpp
    src/input_output/FGScript.cpp
    src/input_output/FGGroundCallback.cpp
    src/input_output/FGXMLElement.cpp
    src/input_output/FGPropertyManager.cpp

    #src/simgear/xml/xmltok_impl.c
    #src/simgear/xml/easyxml.cpp
    src/simgear/xml/xmlrole.c
    src/simgear/xml/easyxml.cxx
    src/simgear/xml/xmltok.c
    #src/simgear/xml/xmltok_ns.c
    src/simgear/xml/xmlparse.c
    src/simgear/props/props.cxx
    src/simgear/magvar/coremag.cxx

    src/math/FGQuaternion.cpp
    src/math/FGColumnVector3.cpp
    src/math/FGStateSpace.cpp
    src/math/FGPropertyValue.cpp
    src/math/FGRungeKutta.cpp
    src/math/FGRealValue.cpp
    src/math/FGModelFunctions.cpp
    src/math/FGFunction.cpp
    src/math/FGNelderMead.cpp
    src/math/FGTable.cpp
    src/math/FGLocation.cpp
    src/math/FGMatrix33.cpp
    src/math/FGCondition.cpp

    src/models/FGAccelerations.cpp
    src/models/flight_control/FGAccelerometer.cpp
    src/models/flight_control/FGSwitch.cpp
    src/models/flight_control/FGPID.cpp
    src/models/flight_control/FGFCSFunction.cpp
    src/models/flight_control/FGSummer.cpp
    src/models/flight_control/FGDeadBand.cpp
    src/models/flight_control/FGGyro.cpp
    src/models/flight_control/FGActuator.cpp
    src/models/flight_control/FGFCSComponent.cpp
    src/models/flight_control/FGMagnetometer.cpp
    src/models/flight_control/FGFilter.cpp
    src/models/flight_control/FGGain.cpp
    src/models/flight_control/FGSensor.cpp
    src/models/flight_control/FGKinemat.cpp
    src/models/FGBuoyantForces.cpp
    src/models/FGExternalForce.cpp
    src/models/FGLGear.cpp
    src/models/FGInput.cpp
    src/models/FGInertial.cpp
    src/models/FGFCS.cpp
    src/models/FGPropagate.cpp
    src/models/FGAuxiliary.cpp
    src/models/FGPropulsion.cpp
    src/models/FGOutput.cpp
    src/models/FGGasCell.cpp
    src/models/FGMassBalance.cpp
    src/models/FGAtmosphere.cpp
    src/models/atmosphere/FGStandardAtmosphere.cpp
    src/models/atmosphere/FGWinds.cpp
    src/models/atmosphere/FGMSISData.cpp
    src/models/atmosphere/FGMSIS.cpp
    src/models/atmosphere/FGMars.cpp
    src/models/FGAircraft.cpp
    src/models/FGExternalReactions.cpp
    src/models/FGGroundReactions.cpp
    src/models/propulsion/FGTank.cpp
    src/models/propulsion/FGForce.cpp
    src/models/propulsion/FGThruster.cpp
    src/models/propulsion/FGPropeller.cpp
    src/models/propulsion/FGTurbine.cpp
    src/models/propulsion/FGElectric.cpp
    src/models/propulsion/FGRotor.cpp
    src/models/propulsion/FGPiston.cpp
    src/models/propulsion/FGTurboProp.cpp
    src/models/propulsion/FGTransmission.cpp
    src/models/propulsion/FGNozzle.cpp
    src/models/propulsion/FGRocket.cpp
    src/models/propulsion/FGEngine.cpp
    src/models/FGAerodynamics.cpp
    src/models/FGModel.cpp

    src/initialization/FGInitialCondition.cpp
    src/initialization/FGSimplexTrim.cpp
    #src/initialization/FGTrimAnalysisControl.cpp
    #src/initialization/FGTrimAnalysis.cpp
    src/initialization/FGTrimAxis.cpp
    src/initialization/FGTrim.cpp
    src/initialization/FGTrimmer.cpp

    src/FGFDMExec.cpp
    src/FGJSBBase.cpp
    )

# public headers
set(JSBSIM_PRIVATE_HDRS
    )

# private headers
set(JSBSIM_PRIVATE_HDRS
    )

# link libraries
set(JSBSIM_LINK_LIBRARIES
    )
if (MINGW)
    list(APPEND JSBSIM_LINK_LIBRARIES
        wsock32
        ws2_32
        )
endif()

# static library
add_library(jsbsimStatic STATIC ${JSBSIM_SRCS} ${JSBSIM_PUBLIC_HDRS} ${JSBSIM_PRIVATE_HDRS})
set(STATIC_LIBNAME_SUFFIX "")
if (MSVC)
    set(STATIC_LIBNAME_SUFFIX "static")
endif()
set_target_properties(jsbsimStatic PROPERTIES 
    OUTPUT_NAME jsbsim${STATIC_LIBNAME_SUFFIX}
    VERSION ${LIBRARY_VERSION} 
    PUBLIC_HEADER "${JSBSIM_PUBLIC_HDRS}"
    PRIVATE_HEADER "${JSBSIM_PRIVATE_HDRS}"
    )
target_link_libraries(jsbsimStatic ${JSBSIM_LINK_LIBRARIES})
install(TARGETS jsbsimStatic
    ARCHIVE DESTINATION "lib" COMPONENT Dev
    # where to put files on non-mac computers, mac overrides and uses default
    PRIVATE_HEADER DESTINATION "include/jsbsim" COMPONENT Runtime
    PUBLIC_HEADER DESTINATION "include/jsbsim" COMPONENT Runtime
    )

# shared library
if(WITH_BUILD_SHARED)
    add_library(jsbsimShared SHARED ${JSBSIM_SRCS} ${JSBSIM_PUBLIC_HDRS} ${JSBSIM_PRIVATE_HDRS})
    set_target_properties(jsbsimShared PROPERTIES 
        OUTPUT_NAME jsbsim
        VERSION ${LIBRARY_VERSION} 
        SOVERSION ${LIBRARY_SOVERSION}
        FRAMEWORK ON
        PUBLIC_HEADER "${JSBSIM_PUBLIC_HDRS}"
        PRIVATE_HEADER "${JSBSIM_PRIVATE_HDRS}"
        )
    target_link_libraries(jsbsimShared ${JSBSIM_LINK_LIBRARIES})
    install(TARGETS jsbsimShared 
        LIBRARY DESTINATION "lib" COMPONENT Runtime
        FRAMEWORK DESTINATION "/Library/Frameworks" COMPONENT Runtime
        # where to put files on non-mac computers, mac overrides and uses default
        PRIVATE_HEADER DESTINATION "include/jsbsim" COMPONENT Runtime
        PUBLIC_HEADER DESTINATION "include/jsbsim" COMPONENT Runtime
        )
endif()

# command line executable
add_executable(jsbsim-cmd src/JSBSim.cpp)
target_link_libraries(jsbsim-cmd jsbsimStatic)
install(TARGETS jsbsim-cmd
    RUNTIME DESTINATION "bin" COMPONENT Runtime
    )

# gui
if (WITH_GUI)
    add_subdirectory(gui)
endif()

# packaging
if (WIN32)
    # set NSIS image, nsis bug requuires atleast one file with 4 slashes to install correctly
    set(CPACK_PACKAGE_ICON "${CMAKE_SOURCE_DIR}\\\\/cmake/jsbsim-nsis.bmp")
    set(CPACK_NSIS_MUI_ICON "${CMAKE_SOURCE_DIR}/cmake/jsbsim.ico")
    set(CPACK_NSIS_MUI_UNIICON "${CMAKE_SOURCE_DIR}/cmake/jsbsim.ico")
endif()
configure_file("COPYING" "COPYING.txt" COPYONLY) # adding file extensions
configure_file("README" "README.txt" COPYONLY)
set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_BINARY_DIR}/COPYING.txt")
set(CPACK_RESOURCE_FILE_README "${CMAKE_BINARY_DIR}/README.txt")
set(CPACK_PACKAGE_DESCRIPTION_FILE "${CPACK_RESOURCE_FILE_README}")
set(CPACK_RESOURCE_FILE_WELCOME "${CMAKE_SOURCE_DIR}/cmake/WELCOME.txt")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "jsbsim simplex trimming branch")
set(CPACK_PACKAGE_VENDOR ${PROJECT_CONTACT_VENDOR})
set(CPACK_PACKAGE_CONTACT "${PROJECT_CONTACT_EMAIL}")
set(CPACK_PACKAGE_VERSION ${PROJECT_VERSION})
set(CPACK_SET_DESTDIR TRUE)
set(CPACK_COMPONENTS_GROUPING "ALL_COMPONENTS_IN_ONE")
set(CPACK_DEBIAN_PACKAGE_DEPENDS "libstdc++6, libgcc1, libc6")
if (WITH_GUI)
    set(CPACK_DEBIAN_PACKAGE_DEPENDS "${CPACK_DEBIAN_PACKAGE_DEPENDS}, libqtcore4, libqtgui4, libxcb1")
    if (WITH_ARKOSG)
        set(CPACK_DEBIAN_PACKAGE_DEPENDS "${CPACK_DEBIAN_PACKAGE_DEPENDS}, libqt4-opengl, libgl1-mesa-glx")
    endif()
endif()
set(CPACK_NSIS_MODIFY_PATH ON)
set(CPACK_PACKAGE_EXECUTABLES
    #"jsbsim-cmd" "Command Line"
    )
if (WITH_GUI)
    list(APPEND CPACK_PACKAGE_EXECUTABLES
        "jsbsim-gui" "Trimming Gui"
        )
    set(CPACK_CREATE_DESKTOP_LINKS "jsbsim-gui")
endif()
include(CPack)

# pkgconfig
configure_file(pc.in ${PROJECT_NAME}.pc)
install(FILES
    ${PROJECT_BINARY_DIR}/${PROJECT_NAME}.pc
    DESTINATION "lib/pkgconfig" COMPONENT Dev
    )
# vim:sw=4:ts=4:expandtab
