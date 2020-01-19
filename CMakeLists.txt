cmake_minimum_required(VERSION 3.9)
project(abb_libegm)

set(${PROJECT_NAME}_MAJOR_VERSION 1)
set(${PROJECT_NAME}_MINOR_VERSION 0)
set(${PROJECT_NAME}_PATCH_VERSION 0)
set(${PROJECT_NAME}_VERSION ${${PROJECT_NAME}_MAJOR_VERSION}.${${PROJECT_NAME}_MINOR_VERSION}.${${PROJECT_NAME}_PATCH_VERSION})

include(GNUInstallDirs)

if(WIN32)
  set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)
endif()

list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake)

#########################
## Eigen C++ Library   ##
#########################
if(NOT TARGET Eigen3::Eigen)
  find_package(Eigen3 REQUIRED)
endif()

###################################
## Asio Standalone C++ Library   ##
###################################
if(NOT TARGET Asio::Asio)
  find_package(Asio REQUIRED)
endif()

#############################
## Google Protocol Buffers ##
#############################
if(NOT TARGET protobuf::libprotobuf)
  if(WIN32)
    # The protobuf compiler appears to be default installed at ${_CMAKE_INSTALL_DIR}/tools/protobuf,
    # and find_package() doesn't always seem to find it without specifying a hint.
    find_package(Protobuf REQUIRED HINTS ${_CMAKE_INSTALL_DIR}/tools/protobuf)
  else()
    find_package(Protobuf REQUIRED)
  endif()
endif()

option(abb_libegm_CROSSCOMPILING "Enable if you are crosscompiling abb_libegm" OFF)

# Work around Protobuf exporting 'lpthread' as a library: we
# export the dependency on pthread using the configuration files.
list(REMOVE_ITEM Protobuf_LIBRARIES "-lpthread")

# Make sure protoc is present, as apparently the above find_package() doesn't check that.
if(NOT Protobuf_PROTOC_EXECUTABLE AND NOT abb_libegm_CROSSCOMPILING)
  message(FATAL_ERROR "Cannot find required 'protoc', cannot process Protobuf files without it. Aborting.")
endif()

if(NOT abb_libegm_CROSSCOMPILING)
  # Generate C++ for protocol classes (headers and sources
  # get written to the CMAKE_CURRENT_BINARY_DIR location).
  set(EgmProtoFiles egm.proto egm_wrapper.proto egm_wrapper_trajectory.proto)
  if(NOT QUIET)
    message(STATUS "Generating protobuf C++ for: ${EgmProtoFiles}")
  endif()
  protobuf_generate_cpp(EgmProtoSources EgmProtoHeaders ${EgmProtoFiles})
else()
  # TODO: we are hardcoding the generate files for protobuf 3.10 for now)
  # TODO: we should at least check that the protobuf library used is compatible
  # TODO: even better, we should refactor CMake  script to permit to use an
  # TODO: externally defined host's protoc
  set(EgmProtoSources src_crosscompiling/egm.pb.cc
                      src_crosscompiling/egm_wrapper.pb.cc
                      src_crosscompiling/egm_wrapper_trajectory.pb.cc)
endif()

###########
## Build ##
###########
set(
  SRC_FILES
    src/egm_base_interface.cpp
    src/egm_common.cpp
    src/egm_common_auxiliary.cpp
    src/egm_controller_interface.cpp
    src/egm_interpolator.cpp
    src/egm_logger.cpp
    src/egm_udp_server.cpp
    src/egm_trajectory_interface.cpp
    ${EgmProtoSources}
)

add_library(${PROJECT_NAME} STATIC ${SRC_FILES})
target_include_directories(${PROJECT_NAME} PUBLIC
  "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include;${CMAKE_CURRENT_BINARY_DIR}>"
  $<INSTALL_INTERFACE:include>
  PUBLIC ${Boost_INCLUDE_DIRS}
)

if(abb_libegm_CROSSCOMPILING)
  target_include_directories(${PROJECT_NAME} PUBLIC
  "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src_crosscompiling;>")
endif()

target_link_libraries(${PROJECT_NAME}
 ${Boost_LIBRARIES}
 protobuf::libprotobuf
 Asio::Asio
 Eigen3::Eigen
)

# Causes the visibility macros to use dllexport rather than dllimport,
# which is appropriate when building the dll but not consuming it.
target_compile_definitions(${PROJECT_NAME} PRIVATE "ABB_LIBEGM_BUILDING_LIBRARY")
# Use cxx_nullptr to force compilation with C++11 . 
# when CMake >= 3.8 is required, cxx_std_11 feature can be used directly
target_compile_definitions(${PROJECT_NAME} PRIVATE cxx_nullptr)

#############
## Install ##
#############
install(
  DIRECTORY include/
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

install(
  FILES
    ${EgmProtoFiles}
    ${EgmProtoHeaders}
  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/${PROJECT_NAME}
)

install(
  TARGETS ${PROJECT_NAME}
  EXPORT export_${PROJECT_NAME}
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)

# Export targets.
if(NOT abb_libegm_CROSSCOMPILING)
set(export_targets ${export_targets};${PROJECT_NAME})
export(
  EXPORT export_${PROJECT_NAME}
  FILE "${PROJECT_BINARY_DIR}/export_${PROJECT_NAME}.cmake"
)
endif()

# Create the ${PROJECT_NAME}Config.cmake.
set(CONF_INCLUDE_DIRS "${CMAKE_INSTALL_PREFIX}/include")
configure_file(
  ${CMAKE_CURRENT_SOURCE_DIR}/cmake/${PROJECT_NAME}Config.cmake.in
  "${PROJECT_BINARY_DIR}/${CMAKE_FILES_DIRECTORY}/${PROJECT_NAME}Config.cmake" @ONLY
)

# Create the ${PROJECT_NAME}ConfigVersion.cmake.
configure_file(
  ${CMAKE_CURRENT_SOURCE_DIR}/cmake/${PROJECT_NAME}ConfigVersion.cmake.in
  "${PROJECT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake" @ONLY
)

# Create the ${PROJECT_NAME}-extras.cmake.
configure_file(
  ${CMAKE_CURRENT_SOURCE_DIR}/cmake/${PROJECT_NAME}-extras.cmake.in
  "${PROJECT_BINARY_DIR}/${CMAKE_FILES_DIRECTORY}/${PROJECT_NAME}-extras.cmake" @ONLY
)

if(NOT abb_libegm_CROSSCOMPILING)
install(
  FILES
    "${PROJECT_BINARY_DIR}/${CMAKE_FILES_DIRECTORY}/${PROJECT_NAME}Config.cmake"
    "${PROJECT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake"
    "${PROJECT_BINARY_DIR}/${CMAKE_FILES_DIRECTORY}/${PROJECT_NAME}-extras.cmake"
  DESTINATION "share/${PROJECT_NAME}/cmake"
  COMPONENT dev
)

install(
  EXPORT export_${PROJECT_NAME}
  DESTINATION "share/${PROJECT_NAME}/cmake"
  FILE export_${PROJECT_NAME}.cmake
  COMPONENT dev
)
endif()
