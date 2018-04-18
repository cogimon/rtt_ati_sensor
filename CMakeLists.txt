cmake_minimum_required(VERSION 2.8.6)

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release)
endif()


SET(MAJOR_VERSION "1")
SET(MINOR_VERSION "1")
SET(PATCH_VERSION "0")
set(RTT_ATI_SENSOR_VERSION
  ${MAJOR_VERSION}.${MINOR_VERSION}.${PATCH_VERSION})

find_package(catkin QUIET COMPONENTS 
  rtt_ros 
  rtt_roscomm
  roscpp
  std_srvs
  std_msgs
  geometry_msgs
  cmake_modules
)

if(NOT ${catkin_FOUND})
  message(WARNING "[${PROJECT_NAME}] catkin not found. DISABLING all ROS-related parts.")  
endif()

set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/config)

#if(${catkin_FOUND})
#    catkin_package(
#        CATKIN_DEPENDS geometry_msgs std_msgs
#        INCLUDE_DIRS include
#        LIBRARIES ati_sensor
#    )
#endif()

find_package(OROCOS-RTT REQUIRED COMPONENTS rtt-scripting rtt-typekit)
if (NOT OROCOS-RTT_FOUND)
  message (FATAL_ERROR "\nCould not find Orocos-RTT. Please use the shell command\n 'source orocos_toolchain/env.sh' and then run cmake again.")
endif()

include(${OROCOS-RTT_USE_FILE_PATH}/UseOROCOS-RTT.cmake )

# EIGEN 3 HACK
set (Eigen_INCLUDE_DIRS "/usr/include/eigen3")

find_package(orocos_kdl REQUIRED)

find_package(Boost REQUIRED)

#find_package(ati_sensor REQUIRED)
#include_directories(BEFORE SYSTEM ${ati_sensor_INCLUDE_DIRS})
##add_definitions(${ati_sensor_CFLAGS})
#link_directories(${ati_sensor_LIBRARY_DIR} ${ati_sensor_LIBRARY_DIRS})

find_package(RST-RT REQUIRED)
message(STATUS "RST-RT version: ${RST-RT_VERSION}")
include_directories(BEFORE SYSTEM ${RST-RT_INCLUDE_DIRS})
add_definitions(${RST-RT_CFLAGS})
link_directories(${RST-RT_LIBRARY_DIR} ${RST-RT_LIBRARY_DIRS})

orocos_component(${PROJECT_NAME}
  src/rtt_ft_sensor.cpp
  src/ft_sensor.cpp
)

find_package(Xenomai QUIET)
# Allows to build for Xenomai + RTnet
if(XENOMAI_FOUND OR (DEFINED $ENV{OROCOS_TARGET} AND $ENV{OROCOS_TARGET} EQUAL "xenomai"))

    message("[${PROJECT_NAME}] Building ATI FT/Sensor for Xenomai ${XENOMAI_VERSION} + RTnet")
    message("[${PROJECT_NAME}] Compiling with ${XENOMAI_RTDM_LIBRARIES} and ${XENOMAI_RTDM_INCLUDE_DIRS} and ${XENOMAI_RTDM_CFLAGS}")

    # In Xenomai 3 it contains RTnet
    include_directories(${XENOMAI_RTDM_INCLUDE_DIRS})
    add_definitions(${XENOMAI_RTDM_CFLAGS})

    # In Xenomai 2 we need the external RTnet
    if(${XENOMAI_VERSION_MAJOR} EQUAL 2)
        find_package(RTnet REQUIRED)
        include_directories(${RTNET_INCLUDE_DIRS})
    endif()

    set_target_properties(ati_sensor PROPERTIES COMPILE_FLAGS -DXENOMAI_VERSION_MAJOR=${XENOMAI_VERSION_MAJOR})
else()
    message(WARNING "[${PROJECT_NAME}] Building ATI FT/Sensor WITHOUT Xenomai/RTnet support")
    find_package(LibXml2 REQUIRED)
    include_directories(${LIBXML2_INCLUDE_DIR})
endif()

if(${catkin_FOUND})
  include_directories(
    include
    ${catkin_INCLUDE_DIRS}
    ${Eigen_INCLUDE_DIRS}
    ${orocos_kdl_INCLUDE_DIRS}
    ${Boost_INCLUDE_DIRS}
  )

  target_link_libraries(${PROJECT_NAME}
    ${orocos_kdl_LIBRARIES}
    ${catkin_LIBRARIES}
    ${Boost_LIBRARIES}
    ${LIBXML2_LIBRARIES}
    ${XENOMAI_RTDM_LIBRARIES}
  )
else()
  include_directories(
    include
    ${Boost_INCLUDE_DIR}
    ${Eigen_INCLUDE_DIRS}
    ${orocos_kdl_INCLUDE_DIRS}
    ${USE_OROCOS_INCLUDE_DIRS}
    ${RST-RT_INCLUDE_DIRS}
  )

  target_link_libraries(${PROJECT_NAME}
    ${orocos_kdl_LIBRARIES}
    ${Boost_LIBRARIES}
    ${USE_OROCOS_LIBRARIES}
    ${OROCOS-RTT_LIBRARIES}
    ${RST-RT_LIBRARIES}
    ${LIBXML2_LIBRARIES}
    ${XENOMAI_RTDM_LIBRARIES}
  )
endif()

#add_executable(simple_reader_test test/simple_reader.cpp src/ft_sensor.cpp)
#add_executable(gauge_bias_tester  test/test_gauge_bias.cpp src/ft_sensor.cpp)



if(${catkin_FOUND})
  orocos_generate_package(
    INCLUDE_DIRS include
    DEPENDS ${PROJECT_NAME}
    DEPENDS_TARGETS rtt_roscomm
  )
else()
  orocos_generate_package(
    INCLUDE_DIRS include
    DEPENDS ${PROJECT_NAME}
  )
endif()

##############
### Install ##
##############
#
## Mark cpp header files for installation
#install(DIRECTORY include/${PROJECT_NAME}/
  #DESTINATION ${CATKIN_PACKAGE_INCLUDE_DESTINATION}
  #FILES_MATCHING PATTERN "*.hpp"
  #PATTERN ".svn" EXCLUDE
  #)
#  
#install(DIRECTORY launch/
    #DESTINATION ${CATKIN_PACKAGE_SHARE_DESTINATION}/launch
    #PATTERN ".svn" EXCLUDE)
#
#install(FILES scripts/ft_sensor.ops DESTINATION ${CATKIN_PACKAGE_SHARE_DESTINATION}/scripts)
#  
####################
### Documentation ##
####################
#
#
## add a target to generate API documentation with Doxygen
#if(BUILD_DOCUMENTATION)
#find_package(Doxygen)
#if(DOXYGEN_FOUND)
        #configure_file(${CMAKE_CURRENT_SOURCE_DIR}/Doxyfile.in ${CMAKE_CURRENT_BINARY_DIR}/doc/Doxyfile @ONLY)
        #execute_process(
        #COMMAND ${DOXYGEN_EXECUTABLE} ${CMAKE_CURRENT_BINARY_DIR}/doc/Doxyfile
        #WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/doc
        ##COMMENT "Generating API documentation with Doxygen" VERBATIM
        ##DEPENDS ${CMAKE_CURRENT_BINARY_DIR}/doc/Doxyfile
        #)
#else()
#message(FATAL_ERROR
      #"Doxygen is needed to build the documentation. Please install it correctly")
#endif()
#endif()
