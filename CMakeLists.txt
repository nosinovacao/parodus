#   Copyright 2010-2016 Comcast Cable Communications Management, LLC
#
#   Licensed under the Apache License, Version 2.0 (the "License");
#   you may not use this file except in compliance with the License.
#   You may obtain a copy of the License at
#
#       http://www.apache.org/licenses/LICENSE-2.0
#
#   Unless required by applicable law or agreed to in writing, software
#   distributed under the License is distributed on an "AS IS" BASIS,
#   WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
#   See the License for the specific language governing permissions and
#   limitations under the License.

cmake_minimum_required(VERSION 2.8.7)

project(parodus)

include(ExternalProject)
include(CTest)

set(INSTALL_DIR ${CMAKE_CURRENT_BINARY_DIR}/_install)
set(PREFIX_DIR ${CMAKE_CURRENT_BINARY_DIR}/_prefix)
set(INCLUDE_DIR ${INSTALL_DIR}/include)
set(INCLUDE_UCRESOLV ${PREFIX_DIR}/ucresolv/src/ucresolv/include)
set(LIBRARY_DIR ${INSTALL_DIR}/lib)
set(LIBRARY_DIR64 ${INSTALL_DIR}/lib64)
set(COMMON_LIBRARY_DIR ${INSTALL_DIR}/lib/${CMAKE_LIBRARY_ARCHITECTURE})
set(TEST_RESULTS_DIR ${CMAKE_CURRENT_BINARY_DIR}/test_results)
file(MAKE_DIRECTORY ${TEST_RESULTS_DIR})

include_directories(${INCLUDE_DIR}
                    ${INCLUDE_DIR}/cjson
                    ${INCLUDE_DIR}/nopoll
                    ${INCLUDE_DIR}/msgpack
                    ${INCLUDE_DIR}/trower-base64
                    ${INCLUDE_DIR}/wrp-c                   
                    ${INCLUDE_DIR}/libparodus
                    ${INCLUDE_DIR}/cimplog
                    ${INCLUDE_DIR}/cjwt
					)	

# Get git commit hash 
#-------------------------------------------------------------------------------
execute_process(
  COMMAND git describe --tags --always
  WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
  OUTPUT_VARIABLE GIT_COMMIT_TAG
  OUTPUT_STRIP_TRAILING_WHITESPACE
)

add_definitions("-DGIT_COMMIT_TAG=\"${GIT_COMMIT_TAG}\"")	

# Compile options/flags
#-------------------------------------------------------------------------------
add_definitions(-std=c99)
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -D_GNU_SOURCE -DNOPOLL_LOGGER ")

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Werror -Wall -Wno-missing-field-initializers")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Werror -Wall")


# pthread external dependency
#-------------------------------------------------------------------------------
find_package (Threads)

if (NOT BUILD_YOCTO)

# base64 external dependency
#-------------------------------------------------------------------------------
ExternalProject_Add(trower-base64
    PREFIX ${CMAKE_CURRENT_BINARY_DIR}/_prefix/trower-base64
    GIT_REPOSITORY https://github.com/Comcast/trower-base64.git
    GIT_TAG "fbb9440ae2bc1118866baefcea7ff814f16613dd"
    CMAKE_ARGS += -DCMAKE_INSTALL_PREFIX=${INSTALL_DIR} -DBUILD_TESTING=OFF
)
add_library(libtrower-base64 STATIC SHARED IMPORTED)
add_dependencies(libtrower-base64 trower-base64)


# nopoll external dependency
#-------------------------------------------------------------------------------
ExternalProject_Add(nopoll
    PREFIX ${PREFIX_DIR}/nopoll
    GIT_REPOSITORY https://github.com/Comcast/nopoll.git
    GIT_TAG "96569f13d3fd69bc4157ac7951d93e2dc544f9cd"
    CONFIGURE_COMMAND COMMAND <SOURCE_DIR>/autogen.sh --prefix=${PREFIX}
                                                     --includedir=${INCLUDE_DIR}
                                                     --libdir=${LIBRARY_DIR}
                                                     ${CUSTOM_HOST}
    BUILD_IN_SOURCE 1
)
add_library(libnopoll STATIC SHARED IMPORTED)
add_dependencies(libnopoll nopoll)


# nanoMsg external dependency
#-------------------------------------------------------------------------------
ExternalProject_Add(nanomsg
    PREFIX ${CMAKE_CURRENT_BINARY_DIR}/_prefix/nanomsg
    GIT_REPOSITORY https://github.com/nanomsg/nanomsg.git
    GIT_TAG "0c1aa2b288f6b167dbafe7e29c20e6fc7e71c000"
    CMAKE_ARGS += -DCMAKE_INSTALL_PREFIX=${INSTALL_DIR}
)
add_library(libnanomsg STATIC SHARED IMPORTED)
add_dependencies(libnanomsg nanomsg)


# cJSON external dependency
#-------------------------------------------------------------------------------
ExternalProject_Add(cJSON
    #PREFIX ${PREFIX_DIR}/cJSON
    PREFIX ${CMAKE_CURRENT_BINARY_DIR}/_prefix/cJSON
    GIT_REPOSITORY https://github.com/DaveGamble/cJSON.git
    GIT_TAG "aafb64a1c549b7b927e339df6d35b1d5059dc235"
    CMAKE_ARGS += -DCMAKE_INSTALL_PREFIX=${INSTALL_DIR} -DBUILD_TESTING=OFF
)
add_library(libcJSON STATIC SHARED IMPORTED)
add_dependencies(libcJSON cJSON)


# msgpack-c external dependency
#-------------------------------------------------------------------------------
ExternalProject_Add(msgpack
    PREFIX ${CMAKE_CURRENT_BINARY_DIR}/_prefix/msgpack
    GIT_REPOSITORY https://github.com/msgpack/msgpack-c.git
    GIT_TAG "7a98138f27f27290e680bf8fbf1f8d1b089bf138"
    CMAKE_ARGS += -DCMAKE_INSTALL_PREFIX=${INSTALL_DIR}
                  -DMSGPACK_ENABLE_CXX=OFF
                  -DMSGPACK_BUILD_EXAMPLES=OFF
)
add_library(libmsgpack STATIC SHARED IMPORTED)
add_dependencies(libmsgpack msgpack)


# cimplog external dependency
#-------------------------------------------------------------------------------
ExternalProject_Add(cimplog
    PREFIX ${CMAKE_CURRENT_BINARY_DIR}/_prefix/cimplog
    GIT_REPOSITORY https://github.com/Comcast/cimplog.git
    GIT_TAG "8a5fb3c2f182241d17f5342bea5b7688c28cd1fd"
    CMAKE_ARGS += -DCMAKE_INSTALL_PREFIX=${INSTALL_DIR} -DBUILD_TESTING=OFF
)
add_library(libcimplog STATIC SHARED IMPORTED)
add_dependencies(libcimplog cimplog)


# wrp-c external dependency
#-------------------------------------------------------------------------------
ExternalProject_Add(wrp-c
	DEPENDS trower-base64 msgpack cimplog
    PREFIX ${CMAKE_CURRENT_BINARY_DIR}/_prefix/wrp-c
    GIT_REPOSITORY https://github.com/Comcast/wrp-c.git
    GIT_TAG "d42ac3a63b8d6e723995b63e23aa1969e622faa5"
    CMAKE_ARGS += -DCMAKE_INSTALL_PREFIX=${INSTALL_DIR}
                  -DMSGPACK_ENABLE_CXX=OFF
                  -DMSGPACK_BUILD_EXAMPLES=OFF
                  -DBUILD_TESTING=OFF
                  -DMAIN_PROJ_BUILD=ON
                  -DMAIN_PROJ_LIB_PATH=${LIBRARY_DIR}
    		  -DMAIN_PROJ_INCLUDE_PATH=${INCLUDE_DIR}
)
add_library(libwrp-c STATIC SHARED IMPORTED)
add_dependencies(libwrp-c wrp-c)

# libparodus external dependency
#-------------------------------------------------------------------------------
ExternalProject_Add(libparodus
    DEPENDS trower-base64 msgpack nanomsg wrp-c
    PREFIX ${CMAKE_CURRENT_BINARY_DIR}/_prefix/libparodus
    GIT_REPOSITORY https://github.com/Comcast/libparodus.git
    GIT_TAG "a7615d69bd5859e337f467fb7e33182da970a5de"
    CMAKE_ARGS += -DCMAKE_INSTALL_PREFIX=${INSTALL_DIR} -DBUILD_TESTING=OFF
    		  -DMAIN_PROJ_BUILD=ON
    		  -DMAIN_PROJ_LIB_PATH=${LIBRARY_DIR}
    		  -DMAIN_PROJ_LIB64_PATH=${LIBRARY_DIR64}
    		  -DMAIN_PROJ_COMMON_PATH=${COMMON_LIBRARY_DIR}
    		  -DMAIN_PROJ_INCLUDE_PATH=${INCLUDE_DIR}
)
add_library(liblibparodus STATIC SHARED IMPORTED)
add_dependencies(liblibparodus libparodus)

if (ENABLE_SESHAT)
# libseshat external dependency
#-------------------------------------------------------------------------------
ExternalProject_Add(libseshat
    DEPENDS cJSON trower-base64 msgpack nanomsg wrp-c
    PREFIX ${CMAKE_CURRENT_BINARY_DIR}/_prefix/libseshat
    GIT_REPOSITORY https://github.com/comcast/seshat.git
    GIT_TAG "470f8d5e9457755028aae6da65d3df9b62a53942"
    CMAKE_ARGS += -DCMAKE_INSTALL_PREFIX=${INSTALL_DIR} -DBUILD_TESTING=OFF
    	          -DMAIN_PROJ_BUILD=ON
    	          -DMAIN_PROJ_LIB_PATH=${LIBRARY_DIR}
    	          -DMAIN_PROJ_LIB64_PATH=${LIBRARY_DIR64}
    		  -DMAIN_PROJ_COMMON_PATH=${COMMON_LIBRARY_DIR}
    		  -DMAIN_PROJ_INCLUDE_PATH=${INCLUDE_DIR}
)
add_library(liblibseshat STATIC SHARED IMPORTED)
add_dependencies(liblibseshat libseshat)
include_directories(${INCLUDE_DIR}/libseshat)
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DENABLE_SESHAT ")
endif (ENABLE_SESHAT)

# libcjwt external dependency
#-------------------------------------------------------------------------------
ExternalProject_Add(cjwt
    PREFIX ${CMAKE_CURRENT_BINARY_DIR}/_prefix/cjwt
    GIT_REPOSITORY https://github.com/Comcast/cjwt.git
    GIT_TAG "ddd077f0f00407f9657934492b252ee5cfcde535"
    CMAKE_ARGS += -DCMAKE_INSTALL_PREFIX=${INSTALL_DIR} -DBUILD_TESTING=OFF
)
add_library(libcjwt STATIC SHARED IMPORTED)
add_dependencies(libcjwt cjwt)

if (FEATURE_DNS_QUERY)
# libucresolv external dependency
#-------------------------------------------------------------------------------
ExternalProject_Add(ucresolv
    PREFIX ${CMAKE_CURRENT_BINARY_DIR}/_prefix/ucresolv
    GIT_REPOSITORY https://github.com/Comcast/libucresolv.git
    GIT_TAG "b58d49e165208791f84b44e3c079b1b4ef6d5c9d"
    CMAKE_ARGS += -DCMAKE_INSTALL_PREFIX=${INSTALL_DIR}
)
add_library(libucresolv STATIC SHARED IMPORTED)
add_dependencies(libucresolv ucresolv)
include_directories(${INCLUDE_DIR}
                    ${INCLUDE_DIR}/ucresolv
					)		

endif (FEATURE_DNS_QUERY)

if (BUILD_TESTING)
# cmocka external dependency
#-------------------------------------------------------------------------------
ExternalProject_Add(cmocka
    PREFIX ${CMAKE_CURRENT_BINARY_DIR}/_prefix/cmocka
    GIT_REPOSITORY https://github.com/elventear/cmocka.git
    GIT_TAG "b71a3060699bc1a5b00e958be353772f42545ac2"
    CMAKE_ARGS += -DCMAKE_INSTALL_PREFIX=${INSTALL_DIR}
)
add_library(libcmocka STATIC SHARED IMPORTED)
add_dependencies(libcmocka cmocka)

endif (BUILD_TESTING)

endif () # NOT BUILD_YOCTO

if (BUILD_YOCTO)
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DBUILD_YOCTO ")
endif (BUILD_YOCTO)

if (FEATURE_DNS_QUERY)
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DFEATURE_DNS_QUERY ")
endif (FEATURE_DNS_QUERY)

link_directories ( ${LIBRARY_DIR} ${COMMON_LIBRARY_DIR} ${LIBRARY_DIR64} )
add_subdirectory(src)
if (BUILD_TESTING)
    add_subdirectory(tests)
endif (BUILD_TESTING)
