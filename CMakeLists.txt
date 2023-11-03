cmake_minimum_required(VERSION 3.5)
project(akushon_interfaces)

if(NOT CMAKE_C_STANDARD)
  set(CMAKE_C_STANDARD 99)
endif()

if(NOT CMAKE_CXX_STANDARD)
  set(CMAKE_CXX_STANDARD 14)
endif()

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

find_package(ament_cmake REQUIRED)
find_package(rosidl_default_generators REQUIRED)

set(protobuf_MODULE_COMPATIBLE TRUE)
find_package(Protobuf CONFIG REQUIRED)
message(STATUS "Using protobuf ${Protobuf_VERSION}")

set(_PROTOBUF_LIBPROTOBUF protobuf::libprotobuf)
set(_REFLECTION gRPC::grpc++_reflection)
if(CMAKE_CROSSCOMPILING)
  find_program(_PROTOBUF_PROTOC protoc)
else()
  set(_PROTOBUF_PROTOC $<TARGET_FILE:protobuf::protoc>)
endif()

find_package(gRPC CONFIG REQUIRED)
message(STATUS "Using gRPC ${gRPC_VERSION}")

set(_GRPC_GRPCPP gRPC::grpc++)
if(CMAKE_CROSSCOMPILING)
  find_program(_GRPC_CPP_PLUGIN_EXECUTABLE grpc_cpp_plugin)
else()
  set(_GRPC_CPP_PLUGIN_EXECUTABLE $<TARGET_FILE:gRPC::grpc_cpp_plugin>)
endif()

# Proto file
get_filename_component(akushon_proto "./proto/akushon.proto" ABSOLUTE)
get_filename_component(akushon_proto_path "${akushon_proto}" PATH)

# Generated sources
set(akushon_proto_srcs "${CMAKE_CURRENT_BINARY_DIR}/akushon.pb.cc")
set(akushon_proto_hdrs "${CMAKE_CURRENT_BINARY_DIR}/akushon.pb.h")
set(akushon_grpc_srcs "${CMAKE_CURRENT_BINARY_DIR}/akushon.grpc.pb.cc")
set(akushon_grpc_hdrs "${CMAKE_CURRENT_BINARY_DIR}/akushon.grpc.pb.h")
add_custom_command(
      OUTPUT "${akushon_proto_srcs}" "${akushon_proto_hdrs}" "${akushon_grpc_srcs}" "${akushon_grpc_hdrs}"
      COMMAND ${_PROTOBUF_PROTOC}
      ARGS --grpc_out "${CMAKE_CURRENT_BINARY_DIR}"
        --cpp_out "${CMAKE_CURRENT_BINARY_DIR}"
        -I "${akushon_proto_path}"
        --plugin=protoc-gen-grpc="${_GRPC_CPP_PLUGIN_EXECUTABLE}"
        "${akushon_proto}"
      DEPENDS "${akushon_proto}")

add_library(akushon_proto
    ${akushon_proto_srcs}
    ${akushon_proto_hdrs}
    ${akushon_grpc_srcs}
    ${akushon_grpc_hdrs})

target_link_libraries(akushon_proto
  ${_REFLECTION}
  ${_GRPC_GRPCPP}
  ${_PROTOBUF_LIBPROTOBUF}
)

rosidl_generate_interfaces(${PROJECT_NAME}
  "msg/RunAction.msg"
  "msg/Status.msg"
  "srv/GetActions.srv"
  "srv/SaveActions.srv")

ament_export_dependencies(rosidl_default_runtime)
ament_export_targets(akushon_proto HAS_LIBRARY_TARGET)

install (
  DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
  DESTINATION include
)

install (
  TARGETS akushon_proto
  EXPORT akushon_proto
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION lib
  RUNTIME DESTINATION bin
  INCLUDES DESTINATION include
)

ament_package()
