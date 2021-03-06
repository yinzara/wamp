cmake_minimum_required(VERSION 2.8.11)
include(ExternalProject)
project(wamp)
add_definitions(-DWAMP_LIBRARY -DQT_NO_KEYWORDS -DSECRET_WITH_UNSTABLE -DUNICODE)
if(${CMAKE_SYSTEM_NAME} MATCHES "Linux")
    find_package(PkgConfig)
    pkg_check_modules(LIBSECRET libsecret-1 glib-2.0)
    #find_package(LibSecret REQUIRED)
endif()
if(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
    add_definitions(-DUNICODE)
endif()
find_package(msgpack QUIET)
if(msgpack_FOUND)
else()
    ExternalProject_Add(
      msgpack
      GIT_REPOSITORY "https://github.com/msgpack/msgpack-c.git"
      GIT_TAG "master"
      PREFIX msgpack
      CONFIGURE_COMMAND ""
      BUILD_COMMAND ""
      INSTALL_COMMAND ""
      UPDATE_COMMAND ""
      PATCH_COMMAND ""
      CMAKE_ARGS -DMSGPACK_CXX11=ON -DCMAKE_INSTALL_PREFIX=install
      TEST_COMMAND ""
    )
    set(MSGPACK_INCLUDE_DIRS ${PROJECT_BINARY_DIR}/msgpack/src/msgpack/include)
    message(STATUS "MsgPack: " ${MSGPACK_INCLUDE_DIRS})
endif()
file(GLOB_RECURSE DATABASE_SRCS
         RELATIVE ${CMAKE_CURRENT_SOURCE_DIR}
         ${CMAKE_CURRENT_SOURCE_DIR}/*.cpp
         ${CMAKE_CURRENT_SOURCE_DIR}/*.h)
include_directories(./client ./router ${MSGPACK_INCLUDE_DIRS} ${LIBSECRET_INCLUDE_DIRS})
# Find includes in corresponding build directories
set(CMAKE_INCLUDE_CURRENT_DIR ON)
# Instruct CMake to run moc automatically when needed.
set(CMAKE_AUTOMOC ON)

if(DEFINED QTDIR${_lib_suffix})
	list(APPEND CMAKE_PREFIX_PATH "${QTDIR${_lib_suffix}}")
elseif(DEFINED QTDIR)
	list(APPEND CMAKE_PREFIX_PATH "${QTDIR}")
elseif(DEFINED ENV{QTDIR${_lib_suffix}})
	list(APPEND CMAKE_PREFIX_PATH "$ENV{QTDIR${_lib_suffix}}")
elseif(DEFINED ENV{QTDIR})
	list(APPEND CMAKE_PREFIX_PATH "$ENV{QTDIR}")
endif()

set(QT_MIN_VERSION "5.6.0")
find_package(Qt5 ${QT_MIN_VERSION} CONFIG REQUIRED Core Qml)

add_library(wamp SHARED ${DATABASE_SRCS} qmldir)
get_target_property(core_INCLUDE_DIRECTORIES core INCLUDE_DIRECTORIES)
get_target_property(websockets_INCLUDE_DIRECTORIES websockets INCLUDE_DIRECTORIES)
include_directories(${core_INCLUDE_DIRECTORIES} ${websockets_INCLUDE_DIRECTORIES})
set_property(TARGET wamp PROPERTY CXX_STANDARD 14)
add_dependencies(wamp core websockets msgpack)
if(${CMAKE_SYSTEM_NAME} MATCHES "Windows")
    set(ADDITIONAL_LIBS ${ADDITIONAL_LIBS} Advapi32 Secur32)
endif()
if(${CMAKE_SYSTEM_NAME} MATCHES "Linux")
    set(ADDITIONAL_LIBS ${ADDITIONAL_LIBS} ${LIBSECRET_LIBRARIES})
endif()
target_link_libraries(wamp core websockets Qt5::Core Qt5::Qml ${ADDITIONAL_LIBS})
set(WAMP_INSTALL_PATH "plugins/QFlow/Wamp" CACHE PATH "qFlow Wamp Library Install Path")
install (TARGETS wamp DESTINATION ${WAMP_INSTALL_PATH} COMPONENT qflow)
install (FILES qmldir DESTINATION ${WAMP_INSTALL_PATH} COMPONENT qflow)
