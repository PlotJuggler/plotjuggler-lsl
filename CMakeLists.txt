cmake_minimum_required(VERSION 3.7)

project(plotjuggler_lsl)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

set(CMAKE_MODULE_PATH
    ${CMAKE_MODULE_PATH}
    "${CMAKE_CURRENT_SOURCE_DIR}/cmake"
    )

#-------  Include Qt dependencies -------
set(CMAKE_INCLUDE_CURRENT_DIR ON)
set(CMAKE_AUTOMOC ON)
SET(CMAKE_AUTOUIC ON)

find_package(Qt5 REQUIRED COMPONENTS
    Core
    Widgets
    Xml
    Svg )

include_directories(
    ${Qt5Core_INCLUDE_DIRS}
    ${Qt5Widgets_INCLUDE_DIRS}
    ${Qt5Xml_INCLUDE_DIRS}
    ${Qt5Svg_INCLUDE_DIRS}
    liblsl/include )

set(QT_LIBRARIES
    Qt5::Core
    Qt5::Widgets
    Qt5::Xml
    Qt5::Svg )

add_definitions( ${QT_DEFINITIONS} -DQT_PLUGIN )
set( PJ_LIBRARIES ${QT_LIBRARIES} )

#-------------- Compile LSL first ----------------------

find_package(Git QUIET)
if(GIT_FOUND AND EXISTS "${PROJECT_SOURCE_DIR}/.git")
# Update submodules as needed
    option(GIT_SUBMODULE "Check submodules during build" ON)
    if(GIT_SUBMODULE)
        message(STATUS "Submodule update")
        execute_process(COMMAND ${GIT_EXECUTABLE} submodule update --init --recursive
                        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
                        RESULT_VARIABLE GIT_SUBMOD_RESULT)
        if(NOT GIT_SUBMOD_RESULT EQUAL "0")
            message("git submodule update --init failed with ${GIT_SUBMOD_RESULT}, please checkout submodules")
        endif()
    endif()
endif()

if(EXISTS "${PROJECT_SOURCE_DIR}/liblsl/CMakeLists.txt" AND NOT WIN32)
    SET(LSL_ENABLE_TESTING FALSE CACHE BOOL "Build tests and run" FORCE)
    SET(LSL_ENABLE_CPACK FALSE CACHE BOOL "Enable CPack" FORCE)
    SET(LSL_BUILD_STATIC TRUE CACHE BOOL "Build static library" FORCE)
    SET(LSL_BUILD_SHARED FALSE CACHE BOOL "Build shared library" FORCE)
    add_subdirectory( liblsl )
endif()


#--------------------------------------------------------
#-------------- Build with CATKIN (ROS1) ----------------
if( CATKIN_DEVEL_PREFIX OR catkin_FOUND OR CATKIN_BUILD_BINARY_PACKAGE)

    set(COMPILING_WITH_CATKIN 1)
    message(STATUS "COMPILING_WITH_CATKIN")
    add_definitions(-DCOMPILED_WITH_CATKIN)

    find_package(catkin REQUIRED COMPONENTS plotjuggler )
    catkin_package(
        CATKIN_DEPENDS plotjuggler
        )
    include_directories(${catkin_INCLUDE_DIRS})
    list(APPEND ${PJ_LIBRARIES} ${catkin_LIBRARIES} )
    set(PJ_PLUGIN_INSTALL_DIRECTORY ${CATKIN_PACKAGE_BIN_DESTINATION} )

    #-------------- Build with AMENT (ROS2) -----------------
elseif( DEFINED ENV{AMENT_PREFIX_PATH})

    set(COMPILING_WITH_AMENT 1)
    message(STATUS "COMPILING_WITH_CATKIN")
    add_definitions(-DCOMPILED_WITH_AMENT)

    find_package(ament_cmake REQUIRED)
    find_package(plotjuggler REQUIRED)
    set(PJ_PLUGIN_INSTALL_DIRECTORY lib/${PROJECT_NAME} )

    #------------- Build without any ROS support ------------
else()

    find_package(PlotJuggler REQUIRED)
    message(STATUS "PlotJuggler FOUND")

    include_directories(${PlotJuggler_INCLUDE_DIRS})
    list(APPEND ${PJ_LIBRARIES} ${PlotJuggler_LIBRARIES} )
    set(PJ_PLUGIN_INSTALL_DIRECTORY bin )

endif()

#------- Create the libraries -------

QT5_WRAP_UI ( UI_SRC  DataStreamLSL/datastream_lsl.ui  )

add_library(DataStreamLSL SHARED
    DataStreamLSL/streamer.cpp
    DataStreamLSL/datastream_lsl.cpp
    ${UI_SRC} )

target_link_libraries(DataStreamLSL
    ${PJ_LIBRARIES}
    lsl-static
    lslboost
    )


if (COMPILING_WITH_AMENT)
    ament_target_dependencies(DataStreamLSL plotjuggler)
endif()
#------- Install the libraries -------

install(TARGETS DataStreamLSL DESTINATION ${PJ_PLUGIN_INSTALL_DIRECTORY}  )



