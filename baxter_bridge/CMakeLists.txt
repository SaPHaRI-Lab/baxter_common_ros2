cmake_minimum_required(VERSION 3.5)
project(baxter_bridge)

if(NOT CMAKE_CXX_STANDARD)
  set(CMAKE_CXX_STANDARD 17)
endif()
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options(-Wall -Wextra -Wpedantic)
endif()

option(SAFE_CMD "Ensure JointCommand is limited to Position or Velocity" TRUE)

if(EXISTS "/opt/ros/noetic/include/ros/ros.h")
  set(ROS1_ROOT "/opt/ros/noetic")
else()
  message(FATAL_ERROR "Could not find ROS1 headers. Please check your ROS1 install.")
endif()

set(ROSCONSOLE_LOG "rosconsole_log4cxx")

if(SAFE_CMD)
  add_definitions(-DBAXTER_BRIDGE_SAFE_CMD)
endif()

# --- Find ROS2 pkgs ---
find_package(ament_cmake REQUIRED)
find_package(rclcpp REQUIRED)
find_package(sensor_msgs REQUIRED)
find_package(OpenCV REQUIRED)

find_package(baxter_core_msgs REQUIRED)
find_package(baxter_maintenance_msgs REQUIRED)
find_package(diagnostic_msgs REQUIRED)
find_package(robot_state_publisher REQUIRED)
find_package(rosidl_default_generators REQUIRED)
find_package(trajectory_msgs REQUIRED)

# --- Service files ---
rosidl_generate_interfaces(${PROJECT_NAME}
  "srv/Open.srv"
  "srv/Exists.srv"
)
ament_export_dependencies(rosidl_default_runtime)

rosidl_get_typesupport_target(cpp_typesupport_target "${PROJECT_NAME}" "rosidl_typesupport_cpp")

# --- Find ROS1 libraries manually ---
set(ROS1_LIBS roscpp rosconsole roscpp_serialization rostime xmlrpcpp ${ROSCONSOLE_LOG})
foreach(ROS1_LIB ${ROS1_LIBS})
  add_library(${ROS1_LIB} UNKNOWN IMPORTED)
  set_property(TARGET ${ROS1_LIB} PROPERTY IMPORTED_LOCATION "${ROS1_ROOT}/lib/lib${ROS1_LIB}.so")
endforeach()

# --- Include dirs ---
include_directories(
  ${ROS1_ROOT}/include
  ${OpenCV_INCLUDE_DIRS}
)

# --- Executable ---
add_executable(bridge
  src/baxter_bridge.cpp
  src/bridge.cpp
  src/monitor.cpp
  src/factory.cpp
  src/factory_1to2.cpp
  src/factory_2to1.cpp
  src/topic_poller.cpp
)

target_include_directories(bridge PRIVATE
  include
  ros1_msgs
  ${ROS1_ROOT}/include
  ${OpenCV_INCLUDE_DIRS}
)

target_link_libraries(bridge
  ${ROS1_LIBS}
  ${OpenCV_LIBS}
  ${cpp_typesupport_target}
)

ament_target_dependencies(bridge
  rclcpp
  sensor_msgs
  baxter_core_msgs
  baxter_maintenance_msgs
  diagnostic_msgs
  robot_state_publisher
  trajectory_msgs
)

install(TARGETS bridge
  DESTINATION lib/${PROJECT_NAME}
)

# --- Install ---
install(DIRECTORY launch DESTINATION share/${PROJECT_NAME})

ament_package()

