cmake_minimum_required(VERSION 2.8.3)
project(kinesis_manager)
set(KINESIS_MANAGER_VERSION 0.1.0)

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

set(SERVICE_COMPONENTS kinesis)

find_package(aws_common REQUIRED)
find_package(AWSSDK REQUIRED COMPONENTS ${SERVICE_COMPONENTS})
find_package(CURL REQUIRED)

## Add KVS SDK libraries
include(FindKVSSDK.cmake)

#############
## Build ##
#############

add_library(${PROJECT_NAME} SHARED src/stream_definition_provider.cpp 
  src/kinesis_stream_manager.cpp src/default_callbacks.cpp
  src/kinesis_video_stream_interface.cpp src/kinesis_video_producer_interface.cpp)

target_include_directories(${PROJECT_NAME} PRIVATE
  ${aws_common_INCLUDE_DIRS}
  ${AWSSDK_INCLUDE_DIR}
  ${KVSSDK_EXTERNAL_INCLUDE_DIR})
target_include_directories(${PROJECT_NAME} PUBLIC
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
  $<INSTALL_INTERFACE:include>)
target_link_libraries(${PROJECT_NAME} ${AWSSDK_LINK_LIBRARIES} KinesisVideoProducer cproducer log4cplus )

#############
## Tests ##
#############

enable_testing()
# Build tests which perform API calls to AWS and require user configuration set up
if(BUILD_AWS_TESTING)
  add_definitions(-DBUILD_AWS_TESTING)
endif()

find_common_test_packages()

add_common_gtest(test_kinesis_manager
  test/kinesis_manager_test.cpp
)
if(TARGET test_kinesis_manager)
  target_include_directories(test_kinesis_manager PRIVATE
    ${aws_common_INCLUDE_DIRS}
    ${KVSSDK_EXTERNAL_INCLUDE_DIR}
  )
  target_link_libraries(test_kinesis_manager
    ${PROJECT_NAME}
    ${aws_common_LIBRARIES}
    ${GMOCK_LIBRARY}
    pthread
  )
endif()

add_common_gtest(test_stream_subscription_installer
  test/stream_subscription_installer_test.cpp
)
if(TARGET test_stream_subscription_installer)
  target_include_directories(test_stream_subscription_installer PRIVATE
    ${aws_common_INCLUDE_DIRS}
    ${KVSSDK_EXTERNAL_INCLUDE_DIR}
  )
  target_link_libraries(test_stream_subscription_installer
    ${PROJECT_NAME}
    ${GMOCK_LIBRARY}
    pthread
  )
endif()

add_common_gtest(test_default_callbacks
  test/default_callbacks_test.cpp
)
if(TARGET test_default_callbacks)
  target_include_directories(test_default_callbacks PRIVATE
    ${aws_common_INCLUDE_DIRS}
    ${KVSSDK_EXTERNAL_INCLUDE_DIR}
  )
  target_link_libraries(test_default_callbacks
    ${PROJECT_NAME}
    ${aws_common_LIBRARIES}
    ${GMOCK_LIBRARY}
    pthread
  )
endif()

#############
## Install ##
#############

## Mark executables and/or libraries for installation
install(TARGETS ${PROJECT_NAME}
  EXPORT "${PROJECT_NAME}-targets"
  ARCHIVE DESTINATION lib
  LIBRARY DESTINATION lib
  RUNTIME DESTINATION bin
)

## Mark cpp header files for installation
install(
  DIRECTORY include/
  DESTINATION include
)

## Export KVS SDK artifacts
install(
  DIRECTORY ${KVSSDK_EXTERNAL_LIB_DIR}/
  DESTINATION lib/
)

install(
  DIRECTORY ${KVSSDK_EXTERNAL_INCLUDE_DIR}/
  DESTINATION include
)

export(TARGETS ${PROJECT_NAME}
  FILE "${PROJECT_BINARY_DIR}/${PROJECT_NAME}-targets.cmake")
export(PACKAGE ${PROJECT_NAME})

CONFIGURE_FILE(${PROJECT_NAME}Config.cmake.in
  ${PROJECT_BINARY_DIR}/${PROJECT_NAME}Config.cmake @ONLY)
CONFIGURE_FILE(${PROJECT_NAME}ConfigVersion.cmake.in
  ${PROJECT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake @ONLY)

INSTALL(FILES ${PROJECT_BINARY_DIR}/${PROJECT_NAME}Config.cmake DESTINATION share/${PROJECT_NAME}/cmake)
INSTALL(FILES ${PROJECT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake DESTINATION share/${PROJECT_NAME}/cmake)
INSTALL(EXPORT ${PROJECT_NAME}-targets DESTINATION share/${PROJECT_NAME}/cmake)
INSTALL(FILES FindKVSSDK.cmake DESTINATION share/${PROJECT_NAME}/cmake)
