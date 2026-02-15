cmake_minimum_required(VERSION 3.20)
project(avionmesh-cli VERSION 1.0.0 LANGUAGES C CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Option for SimpleBLE (default ON for desktop)
option(USE_SIMPLEBLE "Enable SimpleBLE support" ON)

# Find packages
find_package(PkgConfig)

# SimpleBLE (optional - can use stub implementation)
set(SIMPLEBLE_SRC_DIR ${CMAKE_CURRENT_SOURCE_DIR}/external/simpleble CACHE PATH "Path to SimpleBLE source")
if(USE_SIMPLEBLE)
    # Try pkg-config first (system install)
    pkg_check_modules(SIMPLEBLE_PKG simpleble QUIET)
    if(SIMPLEBLE_PKG_FOUND)
        set(SIMPLEBLE_FOUND TRUE)
        set(SIMPLEBLE_INCLUDE_DIRS ${SIMPLEBLE_PKG_INCLUDE_DIRS})
        set(SIMPLEBLE_LIBRARIES ${SIMPLEBLE_PKG_LIBRARIES})
    elseif(EXISTS ${SIMPLEBLE_SRC_DIR}/simpleble/CMakeLists.txt)
        # Local source build
        add_subdirectory(${SIMPLEBLE_SRC_DIR}/simpleble ${CMAKE_CURRENT_BINARY_DIR}/simpleble)
        set(SIMPLEBLE_FOUND TRUE)
        set(SIMPLEBLE_INCLUDE_DIRS ${SIMPLEBLE_SRC_DIR}/simpleble/include ${SIMPLEBLE_SRC_DIR}/dependencies/external)
        set(SIMPLEBLE_LIBRARIES simpleble)
    endif()
endif()

# nlohmann/json - fetch if not found
find_package(nlohmann_json 3.11.0 QUIET)
if(NOT nlohmann_json_FOUND)
    include(FetchContent)
    FetchContent_Declare(json
        URL https://github.com/nlohmann/json/releases/download/v3.11.3/json.tar.xz
    )
    FetchContent_MakeAvailable(json)
endif()

# Platform threading
find_package(Threads REQUIRED)

# Include directories
include_directories(${CMAKE_CURRENT_SOURCE_DIR}/include)

# recsrmesh library - bundled as submodule
set(RECSRMESH_SRC_DIR ${CMAKE_CURRENT_SOURCE_DIR}/external/recsrmesh CACHE PATH "Path to recsrmesh-cpp")
add_subdirectory(${RECSRMESH_SRC_DIR} ${CMAKE_CURRENT_BINARY_DIR}/recsrmesh)

# avionmesh library
add_library(avionmesh_lib STATIC
    src/types.cpp
    src/commands.cpp
    src/parse.cpp
    src/avionmesh.cpp
)
target_include_directories(avionmesh_lib PUBLIC
    ${CMAKE_CURRENT_SOURCE_DIR}/include
    ${RECSRMESH_SRC_DIR}/include
)
# MbedTLS is found by recsrmesh; link imported targets explicitly
find_package(MbedTLS REQUIRED)
target_link_libraries(avionmesh_lib PUBLIC recsrmesh MbedTLS::mbedcrypto MbedTLS::mbedtls MbedTLS::mbedx509 Threads::Threads)

# CLI executable
add_executable(avionmesh
    cli/src/main.cpp
    cli/src/ble_bridge.cpp
    cli/src/database.cpp
    cli/src/cli.cpp
)

target_include_directories(avionmesh PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/cli/include
)

if(SIMPLEBLE_FOUND)
    target_include_directories(avionmesh PRIVATE ${SIMPLEBLE_INCLUDE_DIRS})
    target_compile_definitions(avionmesh PRIVATE USE_SIMPLEBLE=1)
    target_link_libraries(avionmesh PRIVATE ${SIMPLEBLE_LIBRARIES})
endif()

target_link_libraries(avionmesh PRIVATE
    avionmesh_lib
    recsrmesh
    nlohmann_json::nlohmann_json
    Threads::Threads
)

# Installation
install(TARGETS avionmesh DESTINATION bin)
