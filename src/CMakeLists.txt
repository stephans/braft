if(BUILD_UNIT_TESTS)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DUNIT_TEST")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DUNIT_TEST")
elseif(NOT DEBUG)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DNDEBUG")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DNDEBUG")
endif()

include_directories(${CMAKE_CURRENT_BINARY_DIR})
include_directories(${CMAKE_SOURCE_DIR}/src)

add_library(braft-static STATIC ${SOURCES})
set_property(TARGET braft-static PROPERTY POSITION_INDEPENDENT_CODE ON)

target_link_libraries(braft-static
  gflags
  protobuf::libprotobuf
  leveldb::leveldb
  ZLIB::zlib
  brpc::brpc
  OpenSSL::SSL
  OpenSSL::Crypto
  )

if (BRPC_WITH_GLOG)
  target_link_libraries(braft-static glog::glog)
endif()

set_target_properties(braft-static PROPERTIES OUTPUT_NAME braft CLEAN_DIRECT_OUTPUT ON)

install(TARGETS braft-static
        RUNTIME DESTINATION bin
        LIBRARY DESTINATION lib${LIBSUFFIX}
        ARCHIVE DESTINATION lib${LIBSUFFIX}
        )
