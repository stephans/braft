cmake_minimum_required(VERSION 3.5)

set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake)

include(HunterGate)
HunterGate(
    URL "https://github.com/lindong28/hunter/archive/41665c8f2f50b1d7ffddcf604a4dbd31db01b0da.zip"
    SHA1 2ba5c8298b68a7f868b5a1a133bfcaf08cb4eb60
)

project(braft)

#option(EXAMPLE_LINK_SO "Whether examples are linked dynamically" OFF)
option(BRPC_WITH_GLOG "With glog" OFF)
option(BRPC_BUILD_SHARED "Build shared library" OFF)
option(BRPC_BUILD_TOOLS "Build brpc tools used for debugging" OFF)

set(WITH_GLOG_VAL "0")
if(BRPC_WITH_GLOG)
    set(WITH_GLOG_VAL "1")
endif()

include(FindThreads)
include(FindProtobuf)

if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    # require at least gcc 4.8
    if(CMAKE_CXX_COMPILER_VERSION VERSION_LESS 4.8)
        message(FATAL_ERROR "GCC is too old, please install a newer version supporting C++11")
    endif()
    if(NOT (CMAKE_CXX_COMPILER_VERSION VERSION_LESS 7.0))
        set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wno-aligned-new")
    endif()
elseif(CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
    # require at least clang 3.3
    if(CMAKE_CXX_COMPILER_VERSION VERSION_LESS 3.3)
        message(FATAL_ERROR "Clang is too old, please install a newer version supporting C++11")
    endif()
else()
    message(WARNING "You are using an unsupported compiler! Compilation has only been tested with Clang and GCC.")
endif()

hunter_add_package(ZLIB)
find_package(ZLIB CONFIG REQUIRED)

hunter_add_package(leveldb)
find_package(leveldb CONFIG REQUIRED)

hunter_add_package(OpenSSL)
find_package(OpenSSL REQUIRED)

if(BRPC_WITH_GLOG)
    hunter_add_package(glog)
    find_package(glog CONFIG REQUIRED)
endif()

hunter_add_package(brpc)
find_package(brpc REQUIRED)

hunter_add_package(Protobuf)
find_package(Protobuf CONFIG REQUIRED)

get_target_property(PROTOBUF_PROTOC_EXECUTABLE protobuf::protoc IMPORTED_LOCATION_RELEASE)

# for *.so
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/output/lib)
# for *.a
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/output/lib)

file(GLOB BRAFT_PROTOS "${CMAKE_SOURCE_DIR}/src/braft/*.proto")
foreach(PROTO ${BRAFT_PROTOS})
    get_filename_component(PROTO_WE ${PROTO} NAME_WE)
    list(APPEND PROTO_SRCS "${CMAKE_CURRENT_BINARY_DIR}/braft/${PROTO_WE}.pb.cc")
    execute_process(
        COMMAND protoc ${PROTO_FLAGS}
        --cpp_out=${CMAKE_CURRENT_BINARY_DIR}
        --proto_path=${PROTOBUF_INCLUDE_DIR}
        --proto_path=${CMAKE_SOURCE_DIR}/src
        --proto_path=${CMAKE_SOURCE_DIR}/src/braft/ ${PROTO}
        WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
        ERROR_VARIABLE PROTO_ERROR
        RESULT_VARIABLE PROTO_RESULT
    )
    if (${PROTO_RESULT} EQUAL 0)
    else ()
        message (FATAL_ERROR "Fail to generate cpp of ${PROTO} : ${PROTO_ERROR}")
    endif()
endforeach()

file(GLOB_RECURSE BRAFT_SOURCES "${CMAKE_SOURCE_DIR}/src/braft/*.cpp")
set(SOURCES
    ${BRAFT_SOURCES}
    ${PROTO_SRCS}
    )

include_directories(
    ${CMAKE_SOURCE_DIR}/src
    ${CMAKE_CURRENT_BINARY_DIR}
)

hunter_add_package(gflags)
find_package(gflags CONFIG REQUIRED)

get_target_property(GFLAGS_INCLUDE_PATH gflags INTERFACE_INCLUDE_DIRECTORIES)
execute_process(
    COMMAND bash -c "grep \"namespace [_A-Za-z0-9]\\+ {\" ${GFLAGS_INCLUDE_PATH}/gflags/gflags_declare.h | head -1 | awk '{print $2}' | tr -d '\n'"
    OUTPUT_VARIABLE GFLAGS_NS
)
if(${GFLAGS_NS} STREQUAL "GFLAGS_NAMESPACE")
    execute_process(
        COMMAND bash -c "grep \"#define GFLAGS_NAMESPACE [_A-Za-z0-9]\\+\" ${GFLAGS_INCLUDE_PATH}/gflags/gflags_declare.h | head -1 | awk '{print $3}' | tr -d '\n'"
        OUTPUT_VARIABLE GFLAGS_NS
    )
endif()

execute_process(
    COMMAND bash -c "git rev-parse --short HEAD | tr -d '\n'"
    OUTPUT_VARIABLE BRAFT_REVISION
)

set(CMAKE_CPP_FLAGS "${CMAKE_CPP_FLAGS} -DBRPC_WITH_GLOG=${WITH_GLOG_VAL} -DGFLAGS_NS=${GFLAGS_NS}")
set(CMAKE_CPP_FLAGS "${CMAKE_CPP_FLAGS} -DBTHREAD_USE_FAST_PTHREAD_MUTEX -D__const__= -D_GNU_SOURCE -DUSE_SYMBOLIZE -DNO_TCMALLOC -D__STDC_FORMAT_MACROS -D__STDC_LIMIT_MACROS -D__STDC_CONSTANT_MACROS -DBRAFT_REVISION=\\\"${BRAFT_REVISION}\\\" -D__STRICT_ANSI__")
set(CMAKE_CPP_FLAGS "${CMAKE_CPP_FLAGS} -g ${DEBUG_SYMBOL}")
set(CMAKE_CPP_FLAGS "${CMAKE_CPP_FLAGS} -msse4 -msse4.2")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${CMAKE_CPP_FLAGS} -O2 -pipe -Wall -W -fPIC -fstrict-aliasing -Wno-invalid-offsetof -Wno-unused-parameter -fno-omit-frame-pointer")
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${CMAKE_CPP_FLAGS} -O2 -pipe -Wall -W -fPIC -fstrict-aliasing -Wno-unused-parameter -fno-omit-frame-pointer")

macro(use_cxx11)
if(CMAKE_VERSION VERSION_LESS "3.1.3")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
else()
    set(CMAKE_CXX_STANDARD 11)
    set(CMAKE_CXX_STANDARD_REQUIRED ON)
endif()
endmacro(use_cxx11)

use_cxx11()

add_subdirectory(src)
if(BUILD_UNIT_TESTS)
    add_subdirectory(test)
endif()

if (BRPC_BUILD_TOOLS)
    add_subdirectory(tools)
endif()

file(COPY ${CMAKE_CURRENT_BINARY_DIR}/braft/
        DESTINATION ${CMAKE_CURRENT_BINARY_DIR}/output/include/braft/
        FILES_MATCHING
        PATTERN "*.h"
        PATTERN "*.hpp"
        )
file(COPY ${CMAKE_SOURCE_DIR}/src/
        DESTINATION ${CMAKE_CURRENT_BINARY_DIR}/output/include/
        FILES_MATCHING
        PATTERN "*.h"
        PATTERN "*.hpp"
        )
install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/output/include/
        DESTINATION include
        FILES_MATCHING
        PATTERN "*.h"
        PATTERN "*.hpp"
        )
