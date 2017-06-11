cmake_minimum_required(VERSION 2.8.3)
project(openni2_camera)

find_package(catkin REQUIRED camera_info_manager dynamic_reconfigure image_transport  nodelet sensor_msgs roscpp message_generation)

find_package(Boost REQUIRED COMPONENTS system thread)

option(FILTER "OB Filter library" OFF)

if( ${FILTER} STREQUAL "ON" )
set(obfilter "On")
else()
set(obfilter "Off")
endif()

include(ExternalProject)

ExternalProject_Add(astra_openni2
  PREFIX astra_openni2
  # SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/astra_openni2/OpenNI2
  GIT_REPOSITORY https://github.com/adityadhawale/OpenNI2.git
  GIT_TAG original
  CONFIGURE_COMMAND echo "no need to configure"
  #${CMAKE_CURRENT_SOURCE_DIR}/libantlr/configure --prefix=<INSTALL_DIR>
  BUILD_IN_SOURCE 1
  BUILD_COMMAND make release FILTER=${obfilter}
  INSTALL_DIR openni2
  INSTALL_COMMAND tar -xjf <SOURCE_DIR>/Packaging/Final/OpenNI-Linux-Arm64-2.2.tar.bz2 -C <INSTALL_DIR> --strip 1 && mkdir -p <INSTALL_DIR>/include && ln -fs <INSTALL_DIR>/Include <INSTALL_DIR>/include/openni
)

generate_dynamic_reconfigure_options(cfg/OpenNI2.cfg)
add_service_files(FILES
  GetSerial.srv)
generate_messages()

catkin_package(
  INCLUDE_DIRS include
  LIBRARIES astra_wrapper
  CATKIN_DEPENDS camera_info_manager dynamic_reconfigure image_transport nodelet sensor_msgs roscpp message_runtime
  DEPENDS libastra
)

link_directories(${CMAKE_CURRENT_BINARY_DIR}/openni2/Redist)

include_directories(include
                    ${catkin_INCLUDE_DIRS}
                    ${Boost_INCLUDE_DIRS}
                    ${CMAKE_CURRENT_BINARY_DIR}/openni2/include
                    )


add_library(astra_wrapper
   src/openni2_convert.cpp
   src/openni2_device.cpp
   src/openni2_device_info.cpp
   src/openni2_timer_filter.cpp
   src/openni2_frame_listener.cpp
   src/openni2_device_manager.cpp
   src/openni2_exception.cpp
   src/openni2_video_mode.cpp
)
target_link_libraries(astra_wrapper ${catkin_LIBRARIES} -lOpenNI2 -pthread
  ${Boost_LIBRARIES} )
add_dependencies(astra_wrapper astra_openni2 ${PROJECT_NAME}_generate_messages_cpp)

add_library(openni2_driver_lib
                src/openni2_driver.cpp
)
target_link_libraries(openni2_driver_lib astra_wrapper  -pthread ${catkin_LIBRARIES} ${Boost_LIBRARIES} )
add_dependencies(openni2_driver_lib astra_openni2 ${PROJECT_NAME}_gencfg ${PROJECT_NAME}_generate_messages_cpp)

add_library(openni2_camera_nodelet
   ros/openni2_camera_nodelet.cpp
)
target_link_libraries(openni2_camera_nodelet openni2_driver_lib -pthread ${catkin_LIBRARIES} ${Boost_LIBRARIES} )
add_dependencies(openni2_camera_nodelet astra_openni2 ${PROJECT_NAME}_gencfg)

add_executable(openni2_camera_node
   ros/openni2_camera_node.cpp
)
target_link_libraries(openni2_camera_node openni2_driver_lib -pthread ${catkin_LIBRARIES} ${Boost_LIBRARIES} )
add_dependencies(openni2_camera_node astra_openni2 ${PROJECT_NAME}_gencfg ${PROJECT_NAME}_generate_messages_cpp)

add_executable(openni2_list_devices
   src/list_devices.cpp
)
target_link_libraries(openni2_list_devices astra_wrapper)

add_executable(openni2_test_wrapper test/test_wrapper.cpp )
target_link_libraries(openni2_test_wrapper astra_wrapper ${Boost_LIBRARIES})
if (UNIX AND NOT APPLE)
  add_executable(astra_usb_reset src/usb_reset.c)
  set(ADDITIONAL_EXECUTABLES "astra_usb_reset")
endif()

install(TARGETS astra_wrapper openni2_camera_nodelet openni2_camera_node openni2_list_devices openni2_driver_lib ${ADDITIONAL_EXECUTABLES}
  ARCHIVE DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
  LIBRARY DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}
  RUNTIME DESTINATION ${CATKIN_PACKAGE_BIN_DESTINATION}
  )

install(FILES ${CMAKE_CURRENT_BINARY_DIR}/openni2/Redist/libOpenNI2.so
  DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}/
)

install(DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/openni2/Redist/OpenNI2
  DESTINATION ${CATKIN_PACKAGE_LIB_DESTINATION}/
)

install(DIRECTORY include/${PROJECT_NAME}/
  DESTINATION ${CATKIN_PACKAGE_INCLUDE_DESTINATION}
)

# add xml file
install(FILES openni2_nodelets.xml
        DESTINATION ${CATKIN_PACKAGE_SHARE_DESTINATION}
)
