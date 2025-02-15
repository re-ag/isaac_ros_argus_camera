# Copyright (c) 2021, NVIDIA CORPORATION.  All rights reserved.
#
# NVIDIA CORPORATION and its licensors retain all intellectual property
# and proprietary rights in and to this software, related documentation
# and any modifications thereto.  Any use, reproduction, disclosure or
# distribution of this software and related documentation without an express
# license agreement from NVIDIA CORPORATION is strictly prohibited.

cmake_minimum_required(VERSION 3.5)
project(isaac_ros_argus_camera_stereo)

# Default to C++17
if(NOT CMAKE_CXX_STANDARD)
  set(CMAKE_CXX_STANDARD 17)
endif()

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

execute_process(
  COMMAND uname -m
  COMMAND tr -d '\n'
  OUTPUT_VARIABLE ARCHITECTURE)
message(STATUS "Architecture: ${ARCHITECTURE}")

find_package(rclcpp REQUIRED)
find_package(ament_cmake_auto REQUIRED)
ament_auto_find_build_dependencies()

# Only build Argus nodes on Jetson platform
if(${ARCHITECTURE} STREQUAL "aarch64")
  # Find VPI dependency
  find_package(vpi REQUIRED)

  # stereo camera node
  ament_auto_add_library(stereo_node SHARED src/argus_camera_stereo_node.cpp)

  target_include_directories(
    stereo_node
    PRIVATE /usr/lib/aarch64-linux-gnu/tegra
            /usr/src/jetson_multimedia_api/argus/include
            /usr/src/jetson_multimedia_api/argus/samples/utils
            /usr/src/jetson_multimedia_api/include
            include)

  target_compile_definitions(stereo_node PRIVATE "COMPOSITION_BUILDING_DLL")

  target_link_libraries(
    stereo_node
    /usr/lib/aarch64-linux-gnu/tegra/libnvargus.so
    /usr/lib/aarch64-linux-gnu/tegra/libnvargus_socketclient.so
    /usr/lib/aarch64-linux-gnu/tegra/libnvargus_socketserver.so
    /usr/lib/aarch64-linux-gnu/tegra/libnvbuf_utils.so
    vpi)

  target_compile_definitions(stereo_node PRIVATE "COMPOSITION_BUILDING_DLL")

  rclcpp_components_register_nodes(stereo_node "isaac_ros::argus::StereoNode")
  set(node_plugins
      "${node_plugins}isaac_ros::argus::StereoNode;$<TARGET_FILE:stereo_node>\n"
  )

  # isaac_ros_argus executable
  ament_auto_add_executable(${PROJECT_NAME} src/argus_camera_stereo_main.cpp)

  target_include_directories(${PROJECT_NAME} PUBLIC include)

  target_link_libraries(${PROJECT_NAME} stereo_node
                        ament_index_cpp::ament_index_cpp vpi)

  install(
    TARGETS ${PROJECT_NAME}
    ARCHIVE DESTINATION lib
    LIBRARY DESTINATION lib
    RUNTIME DESTINATION bin)
endif()

if(BUILD_TESTING)
  find_package(ament_lint_auto REQUIRED)

  # Ignore copyright notices since we use custom NVIDIA Isaac ROS Software License
  set(ament_cmake_copyright_FOUND TRUE)

  ament_lint_auto_find_test_dependencies()

  find_package(launch_testing_ament_cmake REQUIRED)

endif()

ament_auto_package()
