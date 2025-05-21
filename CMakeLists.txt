cmake_minimum_required(VERSION 3.5)
project(liboculus)

if( FIPS_CONFIG AND NOT FIPS_IMPORT )
  get_filename_component(FIPS_ROOT_DIR "../fips" ABSOLUTE)
  include("${FIPS_ROOT_DIR}/cmake/fips.cmake")

  fips_setup()
else()
  find_package(catkin QUIET)
endif()

# == Code common to all builds =======================================

list(APPEND CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake)

find_package( Boost REQUIRED COMPONENTS system )
find_package(g3log REQUIRED)

# Actually not using full SDK, just the header files
# in thirdparty/Oculus
#find_package( OculusSDK REQUIRED )

if (FIPS_CONFIG)
  #== fips-specific section ==========================================
  add_compile_options(-std=c++17)

  ## Set global include paths
  fips_include_directories(
    ${CMAKE_CURRENT_SOURCE_DIR}/include
    ${CMAKE_CURRENT_SOURCE_DIR}/thirdparty
  )

  ## Verbose output about ASIO handlers
  #add_definitions( -DBOOST_ASIO_ENABLE_HANDLER_TRACKING )

  fips_add_subdirectory("lib/")

  if( NOT FIPS_IMPORT )
    add_subdirectory("tools/")
  endif()

  if( FIPS_UNITTESTS )
    add_subdirectory( test/ )
  endif()

  fips_finish()

else()
  # == ament/ROS2 section =================================
  if(NOT FIPS_CONFIG)
    find_package(ament_cmake REQUIRED)
    find_package(Boost REQUIRED COMPONENTS system)
    find_package(g3log_ros REQUIRED)

    set(CMAKE_CXX_STANDARD 17)
    set(CMAKE_CXX_STANDARD_REQUIRED ON)
    add_compile_options(-std=c++17)

    include_directories(
      include
      thirdparty
      ${CMAKE_INSTALL_PREFIX}/include/g3log
    )

    set(oculus_SRCS
      lib/DataRx.cpp
      lib/SonarConfiguration.cpp
      lib/SonarStatus.cpp
      lib/StatusRx.cpp
      lib/SonarPlayer.cpp
      lib/OculusMessageHandler.cpp
      lib/SimpleFireMessage.cpp
      lib/IoServiceThread.cpp
    )

    add_library(oculus SHARED ${oculus_SRCS})
    target_link_libraries(oculus
      Boost::system
      g3log_ros::g3log_ros
      g3log
    )
    ament_target_dependencies(oculus Boost g3log_ros)

    install(
      TARGETS oculus
      EXPORT oculus
      ARCHIVE DESTINATION lib
      LIBRARY DESTINATION lib
      RUNTIME DESTINATION bin
    )

    install(
      DIRECTORY include/${PROJECT_NAME}/ thirdparty/
      DESTINATION include
      FILES_MATCHING PATTERN "*.hpp" PATTERN "*.h"
      PATTERN ".git" EXCLUDE
    )

    ament_export_targets(oculus HAS_LIBRARY_TARGET)
    ament_export_include_directories(include)
    ament_export_libraries(oculus)

    ament_package()
    return()
  endif()
endif()