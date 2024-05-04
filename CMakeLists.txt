cmake_minimum_required(VERSION 3.2)
project(hidapi-test)

set(CMAKE_CXX_STANDARD 11)

################ hidapi

set(HIDAPI_ROOT ${CMAKE_BINARY_DIR}/thirdparty/hidapi)
set(HIDAPI_LIB_DIR ${HIDAPI_ROOT}/bin/lib)
set(HIDAPI_INCLUDE_DIR ${HIDAPI_ROOT}/bin/include)
set(HIDAPI_STATIC_LIB_FILE ${HIDAPI_LIB_DIR}/libhidapi-hidraw.a)

include(ExternalProject)

ExternalProject_Add(hidapi_external
                    PREFIX ${HIDAPI_ROOT}
                    GIT_REPOSITORY "https://github.com/signal11/hidapi.git"
                    GIT_TAG "40cf516139b5b61e30d9403a48db23d8f915f52c"
                    UPDATE_COMMAND ""
                    PATCH_COMMAND ""
                    BINARY_DIR ${HIDAPI_ROOT}/src/hidapi
                    SOURCE_DIR ${HIDAPI_ROOT}/src/hidapi
                    INSTALL_DIR ${HIDAPI_ROOT}/bin
                    CONFIGURE_COMMAND ./configure --prefix=<INSTALL_DIR>
                    BUILD_COMMAND make
                    BUILD_BYPRODUCTS ${HIDAPI_STATIC_LIB_FILE})

# hidapi requires an initial execution of "./bootstrap" to run autoreconf
ExternalProject_Add_Step(hidapi_external
                         bootstrap
                         COMMAND ./bootstrap
                         DEPENDEES download
                         DEPENDERS configure
                         WORKING_DIRECTORY ${HIDAPI_ROOT}/src/hidapi)

add_library(libhidapi STATIC IMPORTED)
set_target_properties(libhidapi PROPERTIES IMPORTED_LOCATION ${HIDAPI_STATIC_LIB_FILE})
add_dependencies(libhidapi hidapi_external)

################ test app

if(APPLE)
  find_library(IOKIT NAMES IOKit)
  find_library(COREFOUNDATION NAMES CoreFoundation)
  set(LIB_DEPS ${IOKIT} ${COREFOUNDATION})
elseif(UNIX)
  # specific for Linux
  message(STATUS "Detected Unix.  Will link with libudev")
  set(LIB_DEPS "-ludev")
else()
  message(FATAL_ERROR "Unsupported platform.  Exiting...")
endif()

include_directories(${HIDAPI_INCLUDE_DIR})
add_executable(hidapi-test src/main.cpp)
target_link_libraries(hidapi-test ${LIB_DEPS} libhidapi)
