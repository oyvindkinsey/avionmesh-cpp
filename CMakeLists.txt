cmake_minimum_required(VERSION 3.20)
project(avionmesh-cli VERSION 1.0.0 LANGUAGES C CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Option for SimpleBLE (default ON for desktop)
option(USE_SIMPLEBLE "Enable SimpleBLE support" ON)

# Find packages
find_package(PkgConfig)

# SimpleBLE (optional - can use stub implementation)
if(USE_SIMPLEBLE)
    pkg_check_modules(SIMPLEBLE simpleble)
    if(SIMPLEBLE_FOUND)
        set(SIMPLEBLE_SUPPORT TRUE)
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

# recsrmesh library - include as subdirectory
set(RECSRMESH_SRC_DIR ${CMAKE_CURRENT_SOURCE_DIR}/../recsrmesh-cpp CACHE PATH "Path to recsrmesh-cpp")
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
target_link_libraries(avionmesh_lib PUBLIC recsrmesh ${MBEDTLS_LIBRARIES} Threads::Threads)

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
    target_link_libraries(avionmesh ${SIMPLEBLE_LIBRARIES})
endif()

target_link_libraries(avionmesh
    avionmesh_lib
    recsrmesh
    nlohmann_json::nlohmann_json
    Threads::Threads
)

# Platform-specific libraries
if(UNIX AND NOT APPLE)
    target_link_libraries(avionmesh pthread)
endif()

# Installation
install(TARGETS avionmesh DESTINATION bin)
