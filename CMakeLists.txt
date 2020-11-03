cmake_minimum_required(VERSION 3.0)
PROJECT(libindi C CXX)

# As moc files are generated in the binary dir, tell CMake
# to always look for includes there:
set(CMAKE_INCLUDE_CURRENT_DIR ON)

LIST(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake_modules/")
LIST(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/../cmake_modules/")
include(GNUInstallDirs)
include(FeatureSummary)
if(ANDROID OR "${CMAKE_SYSTEM_NAME}" STREQUAL "Android")
  set(ANDROID ON)
  add_definitions(-DANDROID)
endif()

include(CMakeCommon)

# Clang Format support
IF (UNIX OR APPLE)
    SET(FORMAT_CODE OFF CACHE BOOL "Enable Clang Format")
    IF (FORMAT_CODE MATCHES ON)
        FILE(GLOB_RECURSE ALL_SOURCE_FILES *.c *.cpp *.h)

        FOREACH(SOURCE_FILE ${ALL_SOURCE_FILES})
            STRING(FIND ${SOURCE_FILE} ${CMAKE_SOURCE_DIR} DIR_FOUND)
            IF (NOT ${DIR_FOUND} EQUAL 0)
                LIST(REMOVE_ITEM ALL_SOURCE_FILES ${SOURCE_FILE})
            ENDIF ()
        ENDFOREACH ()

        FIND_PROGRAM(CLANGFORMAT_EXE NAMES clang-format-5.0)
        IF (CLANGFORMAT_EXE)
            ADD_CUSTOM_TARGET(clang-format COMMAND ${CLANGFORMAT_EXE} -style=file -i ${ALL_SOURCE_FILES})
        ENDIF ()
    ENDIF ()
ENDIF ()

#####################################  INDI version  ################################################
# N.B. DO NOT Forget to update version also in indiapi.h
# Proper way is to use indiversion.h.cmake file but this would break make existing applications so let us stick to the old proven way

set(INDI_SOVERSION "1")
set(CMAKE_INDI_VERSION_MAJOR 1)
set(CMAKE_INDI_VERSION_MINOR 8)
set(CMAKE_INDI_VERSION_RELEASE 7)
set(CMAKE_INDI_VERSION_STRING "${CMAKE_INDI_VERSION_MAJOR}.${CMAKE_INDI_VERSION_MINOR}.${CMAKE_INDI_VERSION_RELEASE}")
set(INDI_VERSION ${CMAKE_INDI_VERSION_MAJOR}.${CMAKE_INDI_VERSION_MINOR}.${CMAKE_INDI_VERSION_RELEASE})

execute_process(
    COMMAND git describe --tags
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
    OUTPUT_VARIABLE GIT_TAG
    OUTPUT_STRIP_TRAILING_WHITESPACE
    RESULT_VARIABLE GIT_TAG_RESULT
)

if (NOT ${GIT_TAG_RESULT} EQUAL 0)
    set(GIT_TAG "${CMAKE_INDI_VERSION_STRING}-tgz")
endif()

add_definitions(-DGIT_TAG_STRING=\"${GIT_TAG}\")

########################################  Paths  ###################################################

set(DATA_INSTALL_DIR "${CMAKE_INSTALL_PREFIX}/share/indi/")
set(BIN_INSTALL_DIR "${CMAKE_INSTALL_PREFIX}/bin")
set(INCLUDE_INSTALL_DIR "${CMAKE_INSTALL_PREFIX}/include")

IF(APPLE)
set(CMAKE_SHARED_LINKER_FLAGS "-undefined dynamic_lookup")
ENDIF(APPLE)

##################################  Install Directories  ###########################################
## the following are directories where stuff will be installed to
set(INCLUDE_INSTALL_DIR      "${CMAKE_INSTALL_PREFIX}/include/")
set(PKGCONFIG_INSTALL_PREFIX "${CMAKE_INSTALL_LIBDIR}/pkgconfig/")
set(UDEVRULES_INSTALL_DIR "/lib/udev/rules.d" CACHE STRING "Base directory for udev rules")

set(PKG_CONFIG_LIBDIR ${CMAKE_INSTALL_PREFIX}/${CMAKE_INSTALL_LIBDIR})

#####################################  Build Options  ##############################################
# Select which components to build and what options to apply

OPTION (INDI_BUILD_SERVER "Build INDI Server" ON)
OPTION (INDI_BUILD_DRIVERS "Build INDI Drivers, Tools, and Examples" ON)
OPTION (INDI_BUILD_CLIENT "Build INDI POSIX Client" OFF)
OPTION (INDI_BUILD_QT5_CLIENT "Build INDI Qt5 Client" OFF)
OPTION (INDI_BUILD_UNITTESTS "Build INDI tests" OFF)
OPTION (INDI_BUILD_WEBSOCKET "Build INDI with Websocket support" OFF)
OPTION (INDI_FAST_BLOB "Build INDI with Fast BLOB support" ON)
OPTION (INDI_CALCULATE_MINMAX "Calculate and store image minimum and maximum values in FITS header" OFF)

###################################################################################################
#########################################  Fast Blob  #############################################
###################################################################################################
IF (INDI_FAST_BLOB)
# Append ENCLEN attribute to outgoing BLOB elements to enable fast parsing by clients
add_definitions(-DWITH_ENCLEN)
ENDIF(INDI_FAST_BLOB)

###################################################################################################
######################################  Calculate Min/Max #########################################
###################################################################################################
IF (INDI_CALCULATE_MINMAX)
# Calculate Min/Max values to store them in FITS header
add_definitions(-DWITH_MINMAX)
ENDIF(INDI_CALCULATE_MINMAX)

###################################################################################################
#####################################  Components  ################################################
###################################################################################################

set_package_properties(Nova PROPERTIES DESCRIPTION "A general purpose, double precision, Celestial Mechanics, Astrometry and Astrodynamics library" URL "http://libnova.sourceforge.net" TYPE REQUIRED PURPOSE "Provides INDI with astrodynamics library.")
set_package_properties(CFITSIO PROPERTIES DESCRIPTION "A library for reading and writing data files in FITS (Flexible Image Transport System) data format" URL "http://heasarc.gsfc.nasa.gov/fitsio/fitsio.html" TYPE REQUIRED PURPOSE "Provides INDI with FITS I/O support.")

####################################################################################################
#
# Component   : INDI Server
# Dependencies: pthreads
# Supported OS: Linux, BSD, MacOS, Cygwin
#
#################################################################################################
if (INDI_BUILD_SERVER)
if (WIN32 OR ANDROID)
    message(WARNING "INDI Server is only supported under Linux, BSD, MacOS, and Cygwin while current system is " ${CMAKE_SYSTEM_NAME})
else()

# 1. Dependencies
find_package(Threads REQUIRED)
# 2. Includes
include_directories( ${CMAKE_CURRENT_SOURCE_DIR})
# 3. Build
SET(indiserver_SRC
    ${CMAKE_CURRENT_SOURCE_DIR}/indiserver.c
    ${CMAKE_CURRENT_SOURCE_DIR}/fq.c
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/lilxml.c)

IF (UNITY_BUILD)
    ENABLE_UNITY_BUILD(indiserver indiserver_SRC 10 c)
ENDIF ()

add_executable(indiserver ${indiserver_SRC})
target_link_libraries(indiserver ${CMAKE_THREAD_LIBS_INIT})
install(TARGETS indiserver RUNTIME DESTINATION bin)
endif (WIN32 OR ANDROID)
endif (INDI_BUILD_SERVER)

#################################################################################################
#
# Component   : INDI Client
# Dependencies: zlib, cfitsio
# Supported OS: Linux, BSD, MacOS, Windows, Cygwin
# N.B. Windows support pending migration of networking code
#################################################################################################
if (INDI_BUILD_CLIENT AND NOT ANDROID)
# 1. Dependencies
find_package(Threads REQUIRED)
find_package(ZLIB REQUIRED)
find_package(CFITSIO REQUIRED)
# 2. Includes
include_directories( ${CMAKE_CURRENT_BINARY_DIR})
include_directories( ${CMAKE_CURRENT_SOURCE_DIR})
include_directories( ${CMAKE_CURRENT_SOURCE_DIR}/libs)
include_directories( ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase)
include_directories( ${ZLIB_INCLUDE_DIR})
include_directories( ${CFITSIO_INCLUDE_DIR})
# 3. Build

SET(indiclient_C_SRC
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/lilxml.c
    ${CMAKE_CURRENT_SOURCE_DIR}/base64.c)

SET(indiclient_CXX_SRC
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/basedevice.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/baseclient.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indiproperty.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indistandardproperty.cpp)

IF (UNITY_BUILD)
    ENABLE_UNITY_BUILD(indiclient_c indiclient_C_SRC 10 c)
    ENABLE_UNITY_BUILD(indiclient_cxx indiclient_CXX_SRC 10 cpp)
ENDIF ()

SET(indiclient_C_SRC ${indiclient_C_SRC}
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/libastro.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indicom.c)

add_library(indiclient STATIC ${indiclient_C_SRC} ${indiclient_CXX_SRC})
if (NOT CYGWIN AND NOT WIN32)
set_target_properties(indiclient PROPERTIES COMPILE_FLAGS "-fPIC")
endif (NOT CYGWIN AND NOT WIN32)
target_link_libraries(indiclient ${CMAKE_THREAD_LIBS_INIT})
install(TARGETS indiclient ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR})
install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/baseclient.h DESTINATION ${INCLUDE_INSTALL_DIR}/libindi COMPONENT Devel)
endif (INDI_BUILD_CLIENT AND NOT ANDROID)

#################################################################################################
#
# Component   : INDI Qt5 Client
# Dependencies: Qt5Network, zlib, cfitsio, Qt5Core
# Supported OS: Linux, BSD, MacOS, Cygwin, Windows, Android
#
#################################################################################################
if (INDI_BUILD_QT5_CLIENT)
set(QT_ANDROID "" CACHE path "Qt Android path")
# Tell CMake to run moc when necessary:
set(CMAKE_AUTOMOC ON)
# 1. Dependencies
if (ANDROID)
    set(Qt5Core_DIR "${QT_ANDROID}/lib/cmake/Qt5Core")
    include(${QT_ANDROID}/lib/cmake/Qt5/Qt5Config.cmake)
    include(${QT_ANDROID}/lib/cmake/Qt5Core/Qt5CoreConfig.cmake)
    include(${QT_ANDROID}/lib/cmake/Qt5Network/Qt5NetworkConfig.cmake)
    if (NOT CFITSIO_DIR)
        message(FATAL_ERROR CFITSIO_DIR must be set)
    else ()
        set(CFITSIO_INCLUDE_DIR ${CFITSIO_DIR})
        set(CFITSIO_LIBRARIES ${CFITSIO_DIR}/libcfitsio.a)
    endif ()
else ()
    find_package(Qt5Network REQUIRED)
    find_package(ZLIB REQUIRED)
    find_package(CFITSIO REQUIRED)
endif ()
# 2. Includes
include_directories( ${CMAKE_CURRENT_BINARY_DIR})
include_directories( ${CMAKE_CURRENT_SOURCE_DIR})
include_directories( ${CMAKE_CURRENT_SOURCE_DIR}/libs)
include_directories( ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase)
include_directories( ${CFITSIO_INCLUDE_DIR})
# 3. Build
message(STATUS "Building INDI Client with Qt5 support")
SET(indiclientqt_C_SRC
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/lilxml.c
    ${CMAKE_CURRENT_SOURCE_DIR}/base64.c)
SET(indiclientqt_CXX_SRC
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/basedevice.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/baseclientqt.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indiproperty.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indistandardproperty.cpp)
IF (UNITY_BUILD)
    ENABLE_UNITY_BUILD(indiclientqt_c indiclientqt_C_SRC 10 c)
    ENABLE_UNITY_BUILD(indiclientqt_cxx indiclientqt_CXX_SRC 10 cpp)
ENDIF ()

SET(indiclientqt_C_SRC ${indiclientqt_C_SRC}
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/libastro.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indicom.c)

add_library(indiclientqt STATIC ${indiclientqt_C_SRC} ${indiclientqt_CXX_SRC})
if (NOT CYGWIN AND NOT WIN32)
set_target_properties(indiclientqt PROPERTIES COMPILE_FLAGS "-fPIC")
endif(NOT CYGWIN AND NOT WIN32)
target_link_libraries(indiclientqt Qt5::Network)
if (WIN32 OR ANDROID)
install(TARGETS indiclientqt ARCHIVE DESTINATION lib)
else(WIN32 OR ANDROID)
install(TARGETS indiclientqt ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR})
endif(WIN32 OR ANDROID)
install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/baseclientqt.h DESTINATION ${INCLUDE_INSTALL_DIR}/libindi COMPONENT Devel)
endif (INDI_BUILD_QT5_CLIENT)

####################################################################################################
#
# Component   : Websocket
# Dependencies: libwebsocketpp, libboost, asio
# Supported OS: Linux, BSD, MacOS, Cygwin
#
#################################################################################################
if (INDI_BUILD_WEBSOCKET)
    find_package(websocketpp REQUIRED)
    find_package(Boost COMPONENTS system thread)
    include_directories (${Boost_INCLUDE_DIRS})
    add_definitions(-DHAVE_WEBSOCKET)
    SET(HAVE_WEBSOCKET 1)
endif (INDI_BUILD_WEBSOCKET)

#################################################################################################
#
# Component   : INDI Drivers, Tools, and Examples
# Dependencies: pthreads, usb1, zLib, cfitsio, nova, curl, jpeg (Linux Only)
# Supported OS: Linux, BSD, MacOS, Cygwin
# N.B. Webcam drivers only supported under Linux (Video4Linux2). Joystick support only under Linux
#
#################################################################################################
if (INDI_BUILD_DRIVERS)
if (WIN32 OR ANDROID)
    message(WARNING "INDI drivers are only supported under Linux, BSD, MacOS, and Cygwin while current system is " ${CMAKE_SYSTEM_NAME})
else(WIN32 OR ANDROID)
# 1. Dependencies
find_package(Threads REQUIRED)
find_package(ZLIB REQUIRED)
find_package(CFITSIO REQUIRED)
find_package(Nova REQUIRED)
find_package(USB1 REQUIRED)
find_package(CURL REQUIRED)
find_package(GSL REQUIRED)
find_package(JPEG REQUIRED)
# Math Library
FIND_LIBRARY(M_LIB m)
# 2. Includes
include_directories( ${CMAKE_CURRENT_BINARY_DIR})
include_directories( ${CMAKE_CURRENT_SOURCE_DIR})
include_directories( ${CMAKE_CURRENT_SOURCE_DIR}/libs)
include_directories( ${CMAKE_CURRENT_SOURCE_DIR}/libs/dsp)
include_directories( ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase)
include_directories( ${CMAKE_CURRENT_SOURCE_DIR}/libs/stream)
include_directories( ${CFITSIO_INCLUDE_DIR})
include_directories( ${NOVA_INCLUDE_DIR})
include_directories( ${USB1_INCLUDE_DIRS})
include_directories( ${GSL_INCLUDE_DIRS})
include_directories( ${JPEG_INCLUDE_DIR} )
IF (${CMAKE_SYSTEM_NAME} MATCHES "Linux|FreeBSD")
    include_directories( ${CMAKE_CURRENT_SOURCE_DIR}/libs/webcam)
ENDIF (${CMAKE_SYSTEM_NAME} MATCHES "Linux|FreeBSD")

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/config-usb.h.cmake ${CMAKE_CURRENT_BINARY_DIR}/config-usb.h)

###################################################################################################
########################################  Sources  ################################################
###################################################################################################
IF (APPLE)
    SET(hidapi_SRCS ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/hid_mac.c)
ELSEIF (WIN32)
    SET(hidapi_SRCS ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/hid_win.c)
ELSE ()
    SET(hidapi_SRCS ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/hid_libusb.c)
ENDIF()

IF (UNIX)
find_package(OggTheora)
IF (OGGTHEORA_FOUND)
INCLUDE_DIRECTORIES(${THEORA_INCLUDE_DIRS})
SET(HAVE_THEORA 1)
SET (theorarecorder_CXX_SRC ${CMAKE_CURRENT_SOURCE_DIR}/libs/stream/recorder/theorarecorder.cpp)
ENDIF(OGGTHEORA_FOUND)

    SET(libstream_CXX_SRC
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/stream/streammanager.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/stream/recorder/recorderinterface.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/stream/recorder/recordermanager.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/stream/recorder/serrecorder.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/stream/encoder/encodermanager.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/stream/encoder/encoderinterface.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/stream/encoder/rawencoder.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/stream/encoder/mjpegencoder.cpp
        ${theorarecorder_CXX_SRC}
        )
    SET(libstream_C_SRC
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/stream/jpegutils.c
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/stream/ccvt_c2.c
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/stream/ccvt_misc.c)
    IF (UNITY_BUILD)
        ENABLE_UNITY_BUILD(libstream libstream_C_SRC 10 c)
        ENABLE_UNITY_BUILD(libstream libstream_CXX_SRC 10 cpp)
    ENDIF ()

    IF (${CMAKE_SYSTEM_NAME} MATCHES "Linux|FreeBSD")
        SET(libwebcam_C_SRC
            ${CMAKE_CURRENT_SOURCE_DIR}/libs/webcam/v4l2_colorspace.c)
        SET(libwebcam_CXX_SRC
            ${CMAKE_CURRENT_SOURCE_DIR}/libs/webcam/v4l2_base.cpp
            ${CMAKE_CURRENT_SOURCE_DIR}/libs/webcam/v4l2_decode/v4l2_decode.cpp
            ${CMAKE_CURRENT_SOURCE_DIR}/libs/webcam/v4l2_decode/v4l2_builtin_decoder.cpp
            )
        IF (UNITY_BUILD)
            ENABLE_UNITY_BUILD(libwebcam libwebcam_C_SRC 10 c)
            ENABLE_UNITY_BUILD(libwebcam libwebcam_CXX_SRC 10 cpp)
        ENDIF (UNITY_BUILD)
    ENDIF (${CMAKE_SYSTEM_NAME} MATCHES "Linux|FreeBSD")
ENDIF(UNIX)

SET(libdsp_C_SRC
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/dsp/align.c
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/dsp/buffer.c
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/dsp/convert.c
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/dsp/fft.c
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/dsp/filters.c
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/dsp/signals.c
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/dsp/convolution.c
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/dsp/stats.c
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/dsp/stream.c
    )

set(fpack_C_SRC
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/fpack/fpack.c
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/fpack/fpackutil.c
    )

SET(indidriver_C_SRC
    ${CMAKE_CURRENT_SOURCE_DIR}/indidriver.c
    ${CMAKE_CURRENT_SOURCE_DIR}/indidrivermain.c
    ${CMAKE_CURRENT_SOURCE_DIR}/eventloop.c
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/lilxml.c
    ${CMAKE_CURRENT_SOURCE_DIR}/base64.c
    )

SET(indidriver_CXX_SRC
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/basedevice.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/defaultdevice.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indiproperty.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indiccd.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indiccdchip.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indisensorinterface.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indicorrelator.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indidetector.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indispectrograph.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/inditelescope.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indifilterwheel.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indifocuserinterface.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indiweatherinterface.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indifocuser.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indirotator.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indiusbdevice.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indiguiderinterface.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indifilterinterface.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indirotatorinterface.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indidome.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indigps.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indiweather.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indidustcapinterface.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indilightboxinterface.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indilogger.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indicontroller.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indistandardproperty.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/connectionplugins/connectioninterface.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/connectionplugins/connectionserial.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/connectionplugins/connectiontcp.cpp
    #${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/connectionplugins/ttybase.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/dsp/manager.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/dsp/dspinterface.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/dsp/transforms.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/dsp/convolution.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/pid/pid.cpp
    )

IF (UNITY_BUILD)
    ENABLE_UNITY_BUILD(indidriver_c indidriver_C_SRC libdsp_C_SRC fpack_C_SRC 10 c)
    ENABLE_UNITY_BUILD(indidriver_cxx indidriver_CXX_SRC 10 cpp)
ENDIF ()

SET(indidriver_C_SRC
    ${indidriver_C_SRC}
    ${libdsp_C_SRC}
    ${fpack_C_SRC}
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/libastro.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indicom.c)

##################################################
########## INDI Default Driver Library ###########
##################################################

if (CYGWIN)
## For Cygwin we only build static library
add_definitions(-U__STRICT_ANSI__)
find_package(FFTW3 REQUIRED)
find_package(Iconv REQUIRED)
add_library(indidriver STATIC ${indidriver_C_SRC} ${indidriver_CXX_SRC} ${libstream_C_SRC} ${libstream_CXX_SRC} ${hidapi_SRCS} ${libdsp_C_SRC} ${fpack_C_SRC})
target_compile_definitions(indidriver PRIVATE "-DHAVE_LIBNOVA")
set_target_properties(indidriver PROPERTIES VERSION ${CMAKE_INDI_VERSION_STRING} SOVERSION ${INDI_SOVERSION} OUTPUT_NAME indidriver)
target_link_libraries(indidriver ${ICONV_LIBRARIES} ${USB1_LIBRARIES} ${NOVA_LIBRARIES} ${CMAKE_THREAD_LIBS_INIT} ${CFITSIO_LIBRARIES} ${M_LIB} ${ZLIB_LIBRARY} ${JPEG_LIBRARY} ${FFTW3_LIBRARIES})
IF (OGGTHEORA_FOUND)
target_link_libraries(indidriver ${OGGTHEORA_LIBRARIES} ${THEORA_LIBRARIES})
ENDIF()
IF (HAVE_WEBSOCKET)
target_link_libraries(indidriver ${Boost_LIBRARIES})
ENDIF()
install(TARGETS indidriver ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR})
else()
## Static indidriver Library
find_package(FFTW3 REQUIRED)
add_library(indidriverstatic STATIC ${indidriver_C_SRC} ${indidriver_CXX_SRC} ${libstream_C_SRC} ${libstream_CXX_SRC} ${libwebcam_C_SRC} ${libwebcam_CXX_SRC} ${hidapi_SRCS} ${libdsp_C_SRC} ${fpack_C_SRC})
set_target_properties(indidriverstatic PROPERTIES COMPILE_FLAGS "-fPIC")
target_compile_definitions(indidriverstatic PRIVATE "-DHAVE_LIBNOVA")
set_target_properties(indidriverstatic PROPERTIES VERSION ${CMAKE_INDI_VERSION_STRING} SOVERSION ${INDI_SOVERSION} OUTPUT_NAME indidriver)
target_link_libraries(indidriverstatic ${USB1_LIBRARIES} ${NOVA_LIBRARIES} ${CMAKE_THREAD_LIBS_INIT} ${CFITSIO_LIBRARIES} ${M_LIB} ${ZLIB_LIBRARY} ${JPEG_LIBRARY} ${FFTW3_LIBRARIES})
IF (OGGTHEORA_FOUND)
target_link_libraries(indidriverstatic ${OGGTHEORA_LIBRARIES} ${THEORA_LIBRARIES})
ENDIF()
IF (HAVE_WEBSOCKET)
target_link_libraries(indidriverstatic ${Boost_LIBRARIES})
ENDIF()
install(TARGETS indidriverstatic ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR})
## Dynamic indidriver Library
find_package(FFTW3 REQUIRED)
add_library(indidriver SHARED ${indidriver_C_SRC} ${indidriver_CXX_SRC} ${libstream_C_SRC} ${libstream_CXX_SRC} ${libwebcam_C_SRC} ${libwebcam_CXX_SRC} ${hidapi_SRCS} ${libdsp_C_SRC} ${fpack_C_SRC})
set_target_properties(indidriver PROPERTIES COMPILE_FLAGS "-fPIC")
target_compile_definitions(indidriver PRIVATE "-DHAVE_LIBNOVA")
set_target_properties(indidriver PROPERTIES VERSION ${CMAKE_INDI_VERSION_STRING} SOVERSION ${INDI_SOVERSION} OUTPUT_NAME indidriver)
target_link_libraries(indidriver ${ICONV_LIBRARIES} ${USB1_LIBRARIES} ${NOVA_LIBRARIES} ${CMAKE_THREAD_LIBS_INIT} ${CFITSIO_LIBRARIES} ${M_LIB} ${ZLIB_LIBRARY} ${JPEG_LIBRARY} ${FFTW3_LIBRARIES})
IF (OGGTHEORA_FOUND)
target_link_libraries(indidriver ${OGGTHEORA_LIBRARIES} ${THEORA_LIBRARIES})
ENDIF()
IF (HAVE_WEBSOCKET)
target_link_libraries(indidriver ${Boost_LIBRARIES})
ENDIF()
#IF (${CMAKE_SYSTEM_NAME} MATCHES "Linux")
#    target_link_libraries(indidriver -lpthread)
#ENDIF ()
install(TARGETS indidriver LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR})
set(PKG_CONFIG_LIBS "${PKG_CONFIG_LIBS} -lindidriver -lindiAlignmentDriver")
endif(CYGWIN)

IF (CMAKE_SYSTEM_NAME MATCHES "FreeBSD")
    # FreeBSD needs to find the correct GNU iconv library.
    find_package(Iconv REQUIRED)
    # The indidriver library is defined earlier in the file before the
    # find has been run. Add an explicit link to avoid runtime errors.
    target_link_libraries(indidriver ${ICONV_LIBRARIES})
ENDIF(CMAKE_SYSTEM_NAME MATCHES "FreeBSD")

##################################################
########### INDI Alignment Subsystem #############
##################################################
add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/alignment)

#####################################
######## AGENT GROUP ################
#####################################

########### Imager ##############
set(imager_SRCS
        ${CMAKE_CURRENT_SOURCE_DIR}/drivers/agent/agent_imager.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/drivers/agent/group.cpp
   )

add_executable(indi_imager_agent ${imager_SRCS})
target_link_libraries(indi_imager_agent indidriver indiclient)
install(TARGETS indi_imager_agent RUNTIME DESTINATION bin)

#################################################################################

#####################################
########## TELESCOPE GROUP ##########
#####################################

####################################################################################################
#
# Component   : LX200 Library
# Dependencies: pthreads
# Supported OS: Linux, BSD, MacOS, Cygwin
#
#################################################################################################
SET(lx200library_SRCS
    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/telescope/lx200driver.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/telescope/lx200telescope.cpp
    )

IF (UNITY_BUILD)
    ENABLE_UNITY_BUILD(lx200library lx200library_SRCS 10 cpp)
ENDIF ()

add_library(indilx200 SHARED ${lx200library_SRCS})
if (NOT CYGWIN AND NOT WIN32)
set_target_properties(indilx200 PROPERTIES COMPILE_FLAGS "-fPIC")
endif (NOT CYGWIN AND NOT WIN32)
target_compile_definitions(indilx200 PRIVATE "-DHAVE_LIBNOVA")
set_target_properties(indilx200 PROPERTIES VERSION ${CMAKE_INDI_VERSION_STRING} SOVERSION ${INDI_SOVERSION} OUTPUT_NAME indilx200)

install(TARGETS indilx200 LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR})
install(FILES
    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/telescope/lx200telescope.h
    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/telescope/lx200driver.h
    DESTINATION ${INCLUDE_INSTALL_DIR}/libindi/mounts COMPONENT Devel)
#################################################################################

########### LX200 Basic #############
SET(lx200basic_SRC
    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/telescope/lx200driver.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/telescope/lx200basic.cpp)

IF (UNITY_BUILD)
    ENABLE_UNITY_BUILD(lx200basic lx200basic_SRC 10 cpp)
ENDIF ()

add_executable(indi_lx200basic ${lx200basic_SRC})

target_link_libraries(indi_lx200basic indidriver)

install(TARGETS indi_lx200basic RUNTIME DESTINATION bin)

#################################################################################

########### LX200 TeenAstro #############

#################################################################################

########### LX200 Generic ###########
SET(lx200generic_SRCS
#    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/telescope/lx200driver.cpp
${CMAKE_CURRENT_SOURCE_DIR}/drivers/telescope/lx200autostar.cpp
#    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/telescope/lx200_16.cpp
#    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/telescope/lx200gps.cpp
#    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/telescope/lx200generic.cpp
#    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/telescope/lx200telescope.cpp
#    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/telescope/lx200classic.cpp
#    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/telescope/lx200apdriver.cpp
#    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/telescope/lx200gemini.cpp
#    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/telescope/lx200zeq25.cpp
#    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/telescope/lx200gotonova.cpp
#    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/telescope/lx200pulsar2.cpp
#    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/telescope/lx200ap_experimentaldriver.cpp
#    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/telescope/lx200ap_experimental.cpp
#    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/telescope/lx200ap.cpp
#    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/telescope/lx200ap_gtocp2.cpp
#    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/telescope/lx200fs2.cpp
#    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/telescope/lx200ss2000pc.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/telescope/lx200_OnStep.cpp
#    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/telescope/lx200_10micron.cpp
#    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/telescope/ioptronHC8406.cpp
#    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/telescope/eq500x.cpp
)

IF (UNITY_BUILD)
     ENABLE_UNITY_BUILD(lx200generic lx200generic_SRCS 10 cpp)
ENDIF ()

add_executable(indi_lx200generic ${lx200generic_SRCS})
 if (CMAKE_SYSTEM_NAME MATCHES "FreeBSD")
   target_compile_definitions(indi_lx200generic PRIVATE -D_XOPEN_SOURCE=700)
ELSE()
   target_compile_definitions(indi_lx200generic PRIVATE -D_XOPEN_SOURCE=600 -D_POSIX_C_SOURCE=200809L)
ENDIF()
target_link_libraries(indi_lx200generic indidriver)

install(TARGETS indi_lx200generic RUNTIME DESTINATION bin )

file(WRITE ${CMAKE_CURRENT_BINARY_DIR}/make_lx200generic_symlink.cmake
"
# exec_program(\"${CMAKE_COMMAND}\" ARGS -E create_symlink indi_lx200generic \$ENV{DESTDIR}${BIN_INSTALL_DIR}/indi_lx200classic)\n
exec_program(\"${CMAKE_COMMAND}\" ARGS -E create_symlink indi_lx200generic \$ENV{DESTDIR}${BIN_INSTALL_DIR}/indi_lx200autostar)\n
# exec_program(\"${CMAKE_COMMAND}\" ARGS -E create_symlink indi_lx200generic \$ENV{DESTDIR}${BIN_INSTALL_DIR}/indi_lx200_16)\n
# exec_program(\"${CMAKE_COMMAND}\" ARGS -E create_symlink indi_lx200generic \$ENV{DESTDIR}${BIN_INSTALL_DIR}/indi_lx200gps)\n
# exec_program(\"${CMAKE_COMMAND}\" ARGS -E create_symlink indi_lx200generic \$ENV{DESTDIR}${BIN_INSTALL_DIR}/indi_lx200ap_experimental)\n
# exec_program(\"${CMAKE_COMMAND}\" ARGS -E create_symlink indi_lx200generic \$ENV{DESTDIR}${BIN_INSTALL_DIR}/indi_lx200ap_gtocp2)\n
# exec_program(\"${CMAKE_COMMAND}\" ARGS -E create_symlink indi_lx200generic \$ENV{DESTDIR}${BIN_INSTALL_DIR}/indi_lx200ap)\n
# exec_program(\"${CMAKE_COMMAND}\" ARGS -E create_symlink indi_lx200generic \$ENV{DESTDIR}${BIN_INSTALL_DIR}/indi_lx200gemini)\n
# exec_program(\"${CMAKE_COMMAND}\" ARGS -E create_symlink indi_lx200generic \$ENV{DESTDIR}${BIN_INSTALL_DIR}/indi_lx200zeq25)\n
# exec_program(\"${CMAKE_COMMAND}\" ARGS -E create_symlink indi_lx200generic \$ENV{DESTDIR}${BIN_INSTALL_DIR}/indi_lx200gotonova)\n
# exec_program(\"${CMAKE_COMMAND}\" ARGS -E create_symlink indi_lx200generic \$ENV{DESTDIR}${BIN_INSTALL_DIR}/indi_lx200pulsar2)\n
# exec_program(\"${CMAKE_COMMAND}\" ARGS -E create_symlink indi_lx200generic \$ENV{DESTDIR}${BIN_INSTALL_DIR}/indi_lx200fs2)\n
# exec_program(\"${CMAKE_COMMAND}\" ARGS -E create_symlink indi_lx200generic \$ENV{DESTDIR}${BIN_INSTALL_DIR}/indi_lx200ss2000pc)\n
exec_program(\"${CMAKE_COMMAND}\" ARGS -E create_symlink indi_lx200generic \$ENV{DESTDIR}${BIN_INSTALL_DIR}/indi_lx200_OnStep)\n
# exec_program(\"${CMAKE_COMMAND}\" ARGS -E create_symlink indi_lx200generic \$ENV{DESTDIR}${BIN_INSTALL_DIR}/indi_lx200_10micron)\n
# exec_program(\"${CMAKE_COMMAND}\" ARGS -E create_symlink indi_lx200generic \$ENV{DESTDIR}${BIN_INSTALL_DIR}/indi_ioptronHC8406)\n
# exec_program(\"${CMAKE_COMMAND}\" ARGS -E create_symlink indi_lx200generic \$ENV{DESTDIR}${BIN_INSTALL_DIR}/indi_eq500x_telescope)\n
")
set_target_properties(indi_lx200generic PROPERTIES POST_INSTALL_SCRIPT ${CMAKE_CURRENT_BINARY_DIR}/make_lx200generic_symlink.cmake)
#################################################################################

########### Celestron GPS ############

#################################################################################

########### Rainbow Mount driver ##########

########### Takahashi Temma ##########

#################################################################################

########### Paramount ##########

#################################################################################

########### Syncscan ###############

########### Syncscan Legacy ###############

########### Sky Commander ###############

########### Generic Digital Setting Circle ###############

########### IEQ Pro Legacy #############

########### IEQ Pro #############

########### IOptronV3 / CEM120 #############

########### Explore Scientific PMC8 #############

########### Telescope Simulator ##############
SET(telescopesimulator_SRC
    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/telescope/telescope_simulator.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/telescope/scopesim_helper.cpp)

add_executable(indi_simulator_telescope ${telescopesimulator_SRC})
target_link_libraries(indi_simulator_telescope indidriver)
install(TARGETS indi_simulator_telescope RUNTIME DESTINATION bin)

########### Telescope Scripting Gateway ##############
SET(telescopescript_SRC
    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/telescope/telescope_script.cpp)

add_executable(indi_script_telescope ${telescopescript_SRC})
target_link_libraries(indi_script_telescope indidriver)
install(TARGETS indi_script_telescope RUNTIME DESTINATION bin)

################ Detector Simulator ################

SET(spectrographsimulator_SRC
    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/spectrograph/spectrograph_simulator.cpp)

add_executable(indi_simulator_spectrograph ${spectrographsimulator_SRC})
target_link_libraries(indi_simulator_spectrograph indidriver)
install(TARGETS indi_simulator_spectrograph RUNTIME DESTINATION bin)

########### CCD Simulator ##############
SET(ccdsimulator_SRC
    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/ccd/ccd_simulator.cpp)

add_executable(indi_simulator_ccd ${ccdsimulator_SRC})
target_link_libraries(indi_simulator_ccd indidriver)
install(TARGETS indi_simulator_ccd RUNTIME DESTINATION bin)

########### Guide Simulator ##############
SET(guidesimulator_SRC
    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/ccd/guide_simulator.cpp)

add_executable(indi_simulator_guide ${guidesimulator_SRC})
target_link_libraries(indi_simulator_guide indidriver)
install(TARGETS indi_simulator_guide RUNTIME DESTINATION bin)

#####################################
########## FOCUSER GROUP ############
#####################################

#################################################################################

################ Focuser Simulator ################

SET(focussimulator_SRC
    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/focuser/focus_simulator.cpp)

add_executable(indi_simulator_focus ${focussimulator_SRC})
target_link_libraries(indi_simulator_focus indidriver)
install(TARGETS indi_simulator_focus RUNTIME DESTINATION bin)

################ Robo Focuser ################

# SET(robofocus_SRC
#     ${CMAKE_CURRENT_SOURCE_DIR}/drivers/focuser/robofocus.cpp)

# add_executable(indi_robo_focus ${robofocus_SRC})
# target_link_libraries(indi_robo_focus indidriver)
# install(TARGETS indi_robo_focus RUNTIME DESTINATION bin)


################ Robo Focuser ################

# SET(fcusb_SRC
#     ${CMAKE_CURRENT_SOURCE_DIR}/drivers/focuser/fcusb.cpp)

# add_executable(indi_fcusb_focus ${fcusb_SRC})
# target_link_libraries(indi_fcusb_focus indidriver)
# install(TARGETS indi_fcusb_focus RUNTIME DESTINATION bin)

################ Rigelsys NFocus Focuser ################

# SET(nfocus_SRC
#     ${CMAKE_CURRENT_SOURCE_DIR}/drivers/focuser/nfocus.cpp)

# add_executable(indi_nfocus ${nfocus_SRC})
# target_link_libraries(indi_nfocus indidriver)
# install(TARGETS indi_nfocus RUNTIME DESTINATION bin)

################ Rigelsys NStep Focuser ################

# SET(nstep_SRC
#     ${CMAKE_CURRENT_SOURCE_DIR}/drivers/focuser/nstep.cpp)

# add_executable(indi_nstep_focus ${nstep_SRC})
# target_link_libraries(indi_nstep_focus indidriver)
# install(TARGETS indi_nstep_focus RUNTIME DESTINATION bin)

################ PlaneWave EFA ################


################ Celestron SCT Focuser ################


################ AAF2 Focuser ################


################ Moonlite Focuser ################

SET(moonlite_SRC
    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/focuser/moonlite.cpp)

add_executable(indi_moonlite_focus ${moonlite_SRC})
target_link_libraries(indi_moonlite_focus indidriver)
install(TARGETS indi_moonlite_focus RUNTIME DESTINATION bin)

################ Moonlite DRO Dual Focuser ################


################ MyFocuserPro2 Focuser ################

SET(myfocuserpro2_SRC
    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/focuser/myfocuserpro2.cpp)

add_executable(indi_myfocuserpro2_focus ${myfocuserpro2_SRC})
target_link_libraries(indi_myfocuserpro2_focus indidriver)
install(TARGETS indi_myfocuserpro2_focus RUNTIME DESTINATION bin)

################ OnFocus Focuser ################


################ Sesto Senso Focuser ################


################ Sesto Senso 2 Focuser ################


################ Televue FocusMaster ################


########### Lakeside ###########


################ Pegasus DMFC Focuser ################


################## USB Focus V3 ##################


################ Microtouch Focuser ################


################ Baader SteelDrive II Focuser ################


################ Baader SteelDrive Focuser ################


################ FocusLynx Focuser ################


################ PerfectStar Focuser ################


################ Starlight Instruments EFS ################


################ hitechfocus Focuser ################


################ DeepSky Dad AF1 ################


################ DeepSky Dad AF2 ################


################ DeepSky Dad AF3 ################


################ JMI Smart Focus Focuser ################


################ Optec TCF-S ################


################ Lacerta MFOC Focuser ################


#################################################################################

#####################################
######## Rotator GROUP #########
#####################################

################ Rotator Simulator ########

SET(rotatorsim_SRC
    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/rotator/rotator_simulator.cpp)

IF (UNITY_BUILD)
    ENABLE_UNITY_BUILD(rotatorsim rotatorsim_SRC 10 cpp)
ENDIF ()

add_executable(indi_simulator_rotator ${rotatorsim_SRC})
target_link_libraries(indi_simulator_rotator indidriver)
install(TARGETS indi_simulator_rotator RUNTIME DESTINATION bin)

################ Optec Gemini Focusing Rotator ########


################ NightCrawler Focusing Rotator ################


################ Pyxis Rotator ################


################ Falcon Rotator ################


################ Integra85 Focusing Rotator ################


#################################################################################

#####################################
######## FILTER WHEEL GROUP #########
#####################################

########### XAGYL Wheel ##############
SET(xagylwheel_SRC
    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/filter_wheel/xagyl_wheel.cpp)

add_executable(indi_xagyl_wheel ${xagylwheel_SRC})
target_link_libraries(indi_xagyl_wheel indidriver)
install(TARGETS indi_xagyl_wheel RUNTIME DESTINATION bin )

########### Filter Simulator ##############
SET(filtersimulator_SRC
    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/filter_wheel/filter_simulator.cpp)

add_executable(indi_simulator_wheel ${filtersimulator_SRC})
target_link_libraries(indi_simulator_wheel indidriver)
install(TARGETS indi_simulator_wheel RUNTIME DESTINATION bin)

########### Manual Filter ##############
SET(manualfilter_SRC
    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/filter_wheel/manual_filter.cpp)

add_executable(indi_manual_wheel ${manualfilter_SRC})
target_link_libraries(indi_manual_wheel indidriver)
install(TARGETS indi_manual_wheel RUNTIME DESTINATION bin)


########## Optec Wheel IFW ############


########## Quantum Wheel ############


########## TruTech Wheel ############


########## QHYCFW Wheel 1 ############


########## QHYCFW Wheel 2 ############


########## QHYCFW Wheel 3 ############


#################################################################################

#####################################
########## DOME GROUP ############
#####################################

################ Dome Simulator ################

SET(domesimulator_SRC
    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/dome/dome_simulator.cpp)

add_executable(indi_simulator_dome ${domesimulator_SRC})
target_link_libraries(indi_simulator_dome indidriver)
install(TARGETS indi_simulator_dome RUNTIME DESTINATION bin)

################ Roll Off ################

SET(rolloff_SRC
    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/dome/roll_off.cpp)

add_executable(indi_rolloff_dome ${rolloff_SRC})
target_link_libraries(indi_rolloff_dome indidriver)
install(TARGETS indi_rolloff_dome RUNTIME DESTINATION bin)

################ Baader Dome ################


################ Dome Pro 2 ##$##############


################ Rigel Dome ################


################ ScopeDome Dome ################


########### Dome Scripting Gateway ##############
SET(domescript_SRC
    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/dome/dome_script.cpp)

add_executable(indi_script_dome ${domescript_SRC})
target_link_libraries(indi_script_dome indidriver)
install(TARGETS indi_script_dome RUNTIME DESTINATION bin)

#################################################################################

#########################################
########### VIDEO GROUP   ###############
#########################################

########### INDI::CCD V4L Driver ###############
IF (${CMAKE_SYSTEM_NAME} MATCHES "Linux|FreeBSD")
    SET(v4l2driverccd_SRC
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/lx/Lx.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/drivers/video/v4l2driver.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/drivers/video/indi_v4l2driver.cpp)

    IF (UNITY_BUILD)
        ENABLE_UNITY_BUILD(v4l2driverccd v4l2driverccd_SRC 10 cpp)
    ENDIF ()

    add_executable(indi_v4l2_ccd ${v4l2driverccd_SRC} ${libwebcam_C_SRC} ${libwebcam_CXX_SRC})
    target_link_libraries(indi_v4l2_ccd ${JPEG_LIBRARY} indidriver)
    install(TARGETS indi_v4l2_ccd RUNTIME DESTINATION bin )

    # For DBK21 camera
    IF (${CMAKE_SYSTEM_NAME} MATCHES "Linux")
        install(FILES ${CMAKE_CURRENT_SOURCE_DIR}/drivers/video/80-dbk21-camera.rules DESTINATION ${UDEVRULES_INSTALL_DIR})
    ENDIF ()
ENDIF ()

#################################################################################

#####################################
############ AUX GROUP ##############
#####################################
IF (${CMAKE_SYSTEM_NAME} MATCHES "Linux")
    INSTALL(FILES ${CMAKE_CURRENT_SOURCE_DIR}/drivers/auxiliary/99-indi_auxiliary.rules DESTINATION ${UDEVRULES_INSTALL_DIR})
ENDIF ()
########### SkySafari Middleware ##############


########### Watch dog ###############
if (INDI_BUILD_CLIENT)
    SET(watchdog_SRC
        ${CMAKE_CURRENT_SOURCE_DIR}/drivers/auxiliary/watchdog.cpp
        ${CMAKE_CURRENT_SOURCE_DIR}/drivers/auxiliary/watchdogclient.cpp)

    IF (UNITY_BUILD)
        ENABLE_UNITY_BUILD(watchdog watchdog_SRC 10 cpp)
    ENDIF ()

    add_executable(indi_watchdog ${watchdog_SRC})

    target_link_libraries(indi_watchdog indidriver indiclient)
    install(TARGETS indi_watchdog RUNTIME DESTINATION bin)
ELSE ()
    MESSAGE(WARNING "Skipping build of INDI WatchDog driver since INDI POSIX Client is not built")
ENDIF ()

########### Flip Flat & Flip Man Driver ###############

SET(flipflat_SRC
    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/auxiliary/flip_flat.cpp)

add_executable(indi_flipflat ${flipflat_SRC})
target_link_libraries(indi_flipflat indidriver)
install(TARGETS indi_flipflat RUNTIME DESTINATION bin)

########### Pegasus Ultimate Power Box Driver ###############


########### Pegasus Pocket Power Box Driver ###############


########### Pegasus Pocket Power Box Advance Driver ###############


########### SnapCap Driver ###############

SET(snapcap_SRC
    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/auxiliary/snapcap.cpp)

add_executable(indi_snapcap ${snapcap_SRC})
target_link_libraries(indi_snapcap indidriver)
install(TARGETS indi_snapcap RUNTIME DESTINATION bin)

########### Light Pollution Meter #############


########### Sky Quality Meter ###############


########### Sky Quality Meter Simulator ###############

SET(sqm_simulator_SRC
    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/auxiliary/sqm_simulator.cpp)

add_executable(indi_simulator_sqm ${sqm_simulator_SRC})
target_link_libraries(indi_simulator_sqm indidriver)
install(TARGETS indi_simulator_sqm RUNTIME DESTINATION bin)

########### Astrometry Driver ###############

SET(astrometry_SRC
    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/auxiliary/astrometrydriver.cpp)

add_executable(indi_astrometry ${astrometry_SRC})
target_link_libraries(indi_astrometry indidriver)
install(TARGETS indi_astrometry RUNTIME DESTINATION bin)

########### STAR2000 Driver ###############


########### Arduino ST4 Driver ###############


########### PlaneWave Delta-T Driver ###############


########### GPUSB Driver ###############


########### Joystick Driver ###############


########### GPS Simulator Driver ###############

SET(gpssimulator_SRC
    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/auxiliary/gps_simulator.cpp)

add_executable(indi_simulator_gps ${gpssimulator_SRC})
target_link_libraries(indi_simulator_gps indidriver)
install(TARGETS indi_simulator_gps RUNTIME DESTINATION bin)

########### USB_Dewpoint Driver ###############


#####################################
########## WEATHER GROUP ############
#####################################

########### Weather Simulator ###############

SET(weathersimulator_SRC
    ${CMAKE_CURRENT_SOURCE_DIR}/drivers/weather/weathersimulator.cpp)

add_executable(indi_simulator_weather ${weathersimulator_SRC})
target_link_libraries(indi_simulator_weather indidriver)
install(TARGETS indi_simulator_weather RUNTIME DESTINATION bin )

########### Weather Meta Driver ###############


########### Weather File Watcher ###############


########### Weather Safety Proxy ###############


########### MBox Driver ###############


########### Vantage Driver ###############


########### OpenWeatherMap Driver ###############

#####################################
############ INDI TOOLS #############
#####################################

########### getINDI ##############
SET(indi_get_SRC
    ${CMAKE_CURRENT_SOURCE_DIR}/eventloop.c
    ${CMAKE_CURRENT_SOURCE_DIR}/base64.c
    ${CMAKE_CURRENT_SOURCE_DIR}/tools/getINDIproperty.c
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/lilxml.c)

IF (UNITY_BUILD)
    ENABLE_UNITY_BUILD(indi_get indi_get_SRC 10 c)
ENDIF ()

SET(indi_get_SRC ${indi_get_SRC}
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/libastro.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indicom.c)

add_executable(indi_getprop ${indi_get_SRC})

target_link_libraries(indi_getprop ${NOVA_LIBRARIES} ${M_LIB} ${ZLIB_LIBRARY})

install(TARGETS indi_getprop RUNTIME DESTINATION bin )

#################################################################################

########### setINDI ##############
SET(indi_set_SRC
    ${CMAKE_CURRENT_SOURCE_DIR}/eventloop.c
    ${CMAKE_CURRENT_SOURCE_DIR}/base64.c
    ${CMAKE_CURRENT_SOURCE_DIR}/tools/setINDIproperty.c
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/lilxml.c)

IF (UNITY_BUILD)
    ENABLE_UNITY_BUILD(indi_set indi_set_SRC 10 c)
ENDIF ()

SET(indi_set_SRC ${indi_set_SRC}
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/libastro.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indicom.c)

add_executable(indi_setprop ${indi_set_SRC})

target_link_libraries(indi_setprop ${NOVA_LIBRARIES} ${M_LIB} ${ZLIB_LIBRARY})

install(TARGETS indi_setprop RUNTIME DESTINATION bin )

#################################################################################

########### evalINDI ##############
SET(indi_eval_SRC
    ${CMAKE_CURRENT_SOURCE_DIR}/eventloop.c
    ${CMAKE_CURRENT_SOURCE_DIR}/base64.c
    ${CMAKE_CURRENT_SOURCE_DIR}/tools/compiler.c
    ${CMAKE_CURRENT_SOURCE_DIR}/tools/evalINDI.c
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/lilxml.c)

IF (UNITY_BUILD)
    ENABLE_UNITY_BUILD(indi_eval indi_eval_SRC 10 c)
ENDIF ()

SET(indi_eval_SRC ${indi_eval_SRC}
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/libastro.cpp
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indicom.c)

add_executable(indi_eval ${indi_eval_SRC})

target_link_libraries(indi_eval ${NOVA_LIBRARIES} ${M_LIB} ${ZLIB_LIBRARY})

install(TARGETS indi_eval RUNTIME DESTINATION bin )

########### HID Test ##############
SET(indi_hid_SRC
    ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/hidtest.cpp
    )

IF (UNITY_BUILD)
    ENABLE_UNITY_BUILD(indi_hid indi_hid_SRC 10 cpp)
ENDIF ()

if(APPLE)
set(LIBS "-framework IOKit -framework CoreFoundation")
elseif(CYGWIN)
set(LIBS ${ICONV_LIBRARIES})
endif(APPLE)

SET(indi_hid_SRC ${indi_hid_SRC} ${hidapi_SRCS})

add_executable(indi_hid_test ${indi_hid_SRC})

target_link_libraries(indi_hid_test ${ICONV_LIBRARIES} ${USB1_LIBRARIES} ${CMAKE_THREAD_LIBS_INIT} ${LIBS})

install(TARGETS indi_hid_test RUNTIME DESTINATION bin )

#################################################################################
## Build Examples

IF (INDI_BUILD_CLIENT)
    add_subdirectory(${CMAKE_CURRENT_SOURCE_DIR}/examples)
ELSE ()
    message(WARNING "Skipping build of examples since INDI POSIX client is not built")
ENDIF ()

#################################################################################

install( FILES drivers.xml ${CMAKE_CURRENT_SOURCE_DIR}/drivers/focuser/indi_tcfs_sk.xml DESTINATION ${DATA_INSTALL_DIR})

configure_file(${CMAKE_CURRENT_SOURCE_DIR}/libindi.pc.cmake ${CMAKE_CURRENT_BINARY_DIR}/libindi.pc @ONLY)
install(FILES ${CMAKE_CURRENT_BINARY_DIR}/libindi.pc DESTINATION ${PKGCONFIG_INSTALL_PREFIX})

if (UNIX)
    INSTALL(FILES
            ${CMAKE_CURRENT_SOURCE_DIR}/libs/stream/streammanager.h
            ${CMAKE_CURRENT_SOURCE_DIR}/libs/stream/jpegutils.h
            ${CMAKE_CURRENT_SOURCE_DIR}/libs/stream/ccvt.h
            ${CMAKE_CURRENT_SOURCE_DIR}/libs/stream/ccvt_types.h
            DESTINATION ${INCLUDE_INSTALL_DIR}/libindi/stream COMPONENT Devel)
    INSTALL(FILES
            ${CMAKE_CURRENT_SOURCE_DIR}/libs/stream/encoder/encodermanager.h
            ${CMAKE_CURRENT_SOURCE_DIR}/libs/stream/encoder/encoderinterface.h
            ${CMAKE_CURRENT_SOURCE_DIR}/libs/stream/encoder/rawencoder.h
            ${CMAKE_CURRENT_SOURCE_DIR}/libs/stream/encoder/mjpegencoder.h
            DESTINATION ${INCLUDE_INSTALL_DIR}/libindi/stream/encoder COMPONENT Devel)
   INSTALL(FILES
            ${CMAKE_CURRENT_SOURCE_DIR}/libs/stream/recorder/recordermanager.h
            ${CMAKE_CURRENT_SOURCE_DIR}/libs/stream/recorder/recorderinterface.h
            ${CMAKE_CURRENT_SOURCE_DIR}/libs/stream/recorder/serrecorder.h
            DESTINATION ${INCLUDE_INSTALL_DIR}/libindi/stream/recorder COMPONENT Devel)
    if (${CMAKE_SYSTEM_NAME} MATCHES "Linux|FreeBSD")
    INSTALL(FILES
            ${CMAKE_CURRENT_SOURCE_DIR}/libs/webcam/v4l2_decode/v4l2_decode.h
            ${CMAKE_CURRENT_SOURCE_DIR}/libs/webcam/v4l2_decode/v4l2_builtin_decoder.h
            ${CMAKE_CURRENT_SOURCE_DIR}/libs/webcam/v4l2_colorspace.h
            DESTINATION ${INCLUDE_INSTALL_DIR}/libindi COMPONENT Devel)
    ENDIF()
ENDIF ()
###################################################################################################
#########################################  Tests  #################################################
###################################################################################################

find_package (GTest)
find_package (GMock)
IF (GTEST_FOUND)
  IF (INDI_BUILD_UNITTESTS)
    MESSAGE (STATUS  "Building unit tests")
    ADD_SUBDIRECTORY(test)
  ELSE (INDI_BUILD_UNITTESTS)
    MESSAGE (STATUS  "Not building unit tests")
  ENDIF (INDI_BUILD_UNITTESTS)
ELSE()
  MESSAGE (STATUS  "GTEST not found, not building unit tests")
ENDIF (GTEST_FOUND)

endif (WIN32 OR ANDROID)
endif(INDI_BUILD_DRIVERS)

###################################################################################################
#######################################  config.h  ################################################
###################################################################################################
# Generate config.h from template
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/config.h.cmake ${CMAKE_CURRENT_BINARY_DIR}/config.h )
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/indiversion.h.cmake ${CMAKE_CURRENT_BINARY_DIR}/indiversion.h )

if (MSVC)
  add_definitions(-D_CRT_SECURE_NO_WARNINGS)
  add_definitions(-D_WINSOCK_DEPRECATED_NO_WARNINGS)
endif()

# Install common dev files for all except server
if (INDI_BUILD_DRIVERS OR INDI_BUILD_CLIENT OR INDI_BUILD_QT5_CLIENT)
    install(FILES ${CMAKE_CURRENT_BINARY_DIR}/indiversion.h DESTINATION ${INCLUDE_INSTALL_DIR}/libindi COMPONENT Devel)
    install( FILES
        ${CMAKE_CURRENT_SOURCE_DIR}/indiapi.h
        ${CMAKE_CURRENT_SOURCE_DIR}/indidevapi.h
        ${CMAKE_CURRENT_SOURCE_DIR}/base64.h
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/lilxml.h
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/libastro.h
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/indicom.h
        ${CMAKE_CURRENT_SOURCE_DIR}/eventloop.h
        ${CMAKE_CURRENT_SOURCE_DIR}/indidriver.h
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/dsp/dsp.h
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/pid/pid.h
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indibase.h
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indibasetypes.h
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/basedevice.h
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/defaultdevice.h
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indiccd.h
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indiccdchip.h
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indisensorinterface.h
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indicorrelator.h
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indidetector.h
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indispectrograph.h
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indifilterwheel.h
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indifocuserinterface.h
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indiweatherinterface.h
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indifocuser.h
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indirotator.h
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/inditelescope.h
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indiguiderinterface.h
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indifilterinterface.h
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indirotatorinterface.h
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indiproperty.h
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indistandardproperty.h
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indidome.h
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indigps.h
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indilightboxinterface.h
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indidustcapinterface.h
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indiweather.h
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indilogger.h
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indicontroller.h
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indiusbdevice.h
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/hidapi.h
        DESTINATION ${INCLUDE_INSTALL_DIR}/libindi COMPONENT Devel)

    if (HAVE_WEBSOCKET)
        install( FILES ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/indiwsserver.h
            DESTINATION ${INCLUDE_INSTALL_DIR}/libindi COMPONENT Devel)
    endif()
    install( FILES
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/connectionplugins/connectioninterface.h
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/connectionplugins/connectionserial.h
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/connectionplugins/connectiontcp.h
        DESTINATION ${INCLUDE_INSTALL_DIR}/libindi/connectionplugins COMPONENT Devel)

    install( FILES
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/dsp/manager.h
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/dsp/dspinterface.h
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/dsp/transforms.h
        ${CMAKE_CURRENT_SOURCE_DIR}/libs/indibase/dsp/convolution.h
        DESTINATION ${INCLUDE_INSTALL_DIR}/libindi/dsp COMPONENT Devel)
    endif (INDI_BUILD_DRIVERS OR INDI_BUILD_CLIENT OR INDI_BUILD_QT5_CLIENT)

feature_summary(WHAT ALL FATAL_ON_MISSING_REQUIRED_PACKAGES)

message(STATUS "The following components are going to be built:")
if (INDI_BUILD_SERVER)
    message(STATUS "## INDI Server")
endif()
if (INDI_BUILD_DRIVERS)
    message(STATUS "## INDI Drivers, Tools, and Examples")
endif()
if (INDI_BUILD_CLIENT)
    message(STATUS "## INDI Client")
endif()
if (INDI_BUILD_QT5_CLIENT)
    message(STATUS "## INDI Qt5 Client")
endif()
