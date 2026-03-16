cmake_minimum_required(VERSION 3.20)
project(5DCUI VERSION 0.0.0 LANGUAGES CXX)

# Extract version from git tags with fallback to CMake project version
find_package(Git QUIET)
if(GIT_FOUND)
    execute_process(
        COMMAND ${GIT_EXECUTABLE} describe --tags --always
        WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
        OUTPUT_VARIABLE GIT_VERSION
        OUTPUT_STRIP_TRAILING_WHITESPACE
        ERROR_QUIET
    )
    
    # Only use git version if we're in a git repo and got a valid tag
    if(GIT_VERSION AND NOT GIT_VERSION STREQUAL "")
        # Remove 'v' prefix if present
        string(REGEX REPLACE "^v" "" GIT_VERSION "${GIT_VERSION}")
        set(PROJECT_VERSION "${GIT_VERSION}")
        message(STATUS "Version from git tag: ${PROJECT_VERSION}")
    else()
        message(STATUS "Not in a git repo or no tags found, using default version: ${PROJECT_VERSION}")
    endif()
else()
    message(STATUS "Git not found, using default version: ${PROJECT_VERSION}")
endif()

# Make all targets PIC by default
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

# Set C++23 standard
set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Print compiler information
message(STATUS "C compiler: ${CMAKE_C_COMPILER}")
message(STATUS "C++ compiler: ${CMAKE_CXX_COMPILER}")
message(STATUS "C compiler ID: ${CMAKE_C_COMPILER_ID}")
message(STATUS "C++ compiler ID: ${CMAKE_CXX_COMPILER_ID}")
message(STATUS "C compiler version: ${CMAKE_C_COMPILER_VERSION}")
message(STATUS "C++ compiler version: ${CMAKE_CXX_COMPILER_VERSION}")

# Compiler-specific options for magic.cpp (inherited by engine core)
if(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    add_compile_options(-Wall -pedantic -Wextra -Wpedantic)
    add_compile_options(-fconstexpr-steps=100000000)
elseif(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    add_compile_options(-Wall -pedantic -Wextra -Wpedantic)
    # GCC equivalent settings
    add_compile_options(
        -fconstexpr-depth=1000000
        -fconstexpr-ops-limit=100000000
    )
elseif(MSVC)
    add_compile_options(/W4 /permissive-)
    add_compile_options(/constexpr:steps100000000)
endif()

# --- 5dchess_engine Core Library Configuration ---

# Engine directory structure
set(ENGINE_ROOT "${CMAKE_SOURCE_DIR}/extern/5dchess_engine")

# Find all engine source files
file(GLOB_RECURSE ENGINE_SOURCES 
    "${ENGINE_ROOT}/src/core/*.cpp"
    "${ENGINE_ROOT}/src/misc/*.cpp"
)

# Find all engine header files
file(GLOB_RECURSE ENGINE_HEADERS 
    "${ENGINE_ROOT}/src/core/*.h"
    "${ENGINE_ROOT}/src/misc/*.h"
    "${ENGINE_ROOT}/src/core/*.inl"
    "${ENGINE_ROOT}/src/misc/*.inl"
)

# Create the core library target
add_library(5dchess_engine_core OBJECT ${ENGINE_SOURCES})

# Add version as compile definition
target_compile_definitions(5dchess_engine_core PUBLIC 
    PROJECT_VERSION_STRING="${PROJECT_VERSION}"
)

# Ensure header files show up in IDEs
target_sources(5dchess_engine_core PRIVATE ${ENGINE_HEADERS})

# Set include directories for the engine
target_include_directories(5dchess_engine_core 
    PUBLIC 
        ${ENGINE_ROOT}/src
        ${ENGINE_ROOT}/src/core
        ${ENGINE_ROOT}/src/misc
)

# --- 5DCUI (SFML Application) Configuration ---

message(STATUS ">>> Compiling SFML-based 5D chess UI (5DCUI)...")

# SFML directory structure
set(SFML_ROOT "${CMAKE_SOURCE_DIR}/extern/SFML3")

# Find all 5DCUI source files (now in project 'src' directory)
file(GLOB_RECURSE SFML_APP_SOURCES 
    "${CMAKE_CURRENT_SOURCE_DIR}/src/*.cpp"
    "${CMAKE_CURRENT_SOURCE_DIR}/src/*.h"
)

# Create the 5DCUI executable
add_executable(5DCUI ${SFML_APP_SOURCES})

# Link against the engine core
target_link_libraries(5DCUI PRIVATE 5dchess_engine_core)

# Set 5DCUI specific include directories
target_include_directories(5DCUI PRIVATE 
    ${CMAKE_CURRENT_SOURCE_DIR}/src
    "${SFML_ROOT}/include"
)

# Set SFML library directories
target_link_directories(5DCUI PRIVATE "${SFML_ROOT}/lib")

# Configure static linking with SFML 3.0.2
target_compile_definitions(5DCUI PRIVATE SFML_STATIC)

# Link SFML libraries based on build type
if(NOT CMAKE_BUILD_TYPE OR CMAKE_BUILD_TYPE STREQUAL "Debug")
    # Link with static debug libraries
    target_link_libraries(5DCUI PRIVATE
        sfml-graphics-s-d
        sfml-window-s-d
        sfml-system-s-d
        freetyped
    )
else()
    # Link with static release libraries
    target_link_libraries(5DCUI PRIVATE
        sfml-graphics-s
        sfml-window-s
        sfml-system-s
        freetype
    )
endif()

# Platform-specific dependencies for static linking
if(WIN32)
    # Windows dependencies
    target_link_libraries(5DCUI PRIVATE
        opengl32.lib
        winmm.lib
        gdi32.lib
        user32.lib
        advapi32.lib
        shell32.lib
    )

    set_target_properties(5DCUI PROPERTIES
        LINK_FLAGS "/SUBSYSTEM:WINDOWS /ENTRY:mainCRTStartup"
    )
    
elseif(APPLE)
    # macOS dependencies
    find_library(COCOA_LIBRARY Cocoa)
    find_library(IOKIT_LIBRARY IOKit)
    find_library(COREFOUNDATION_LIBRARY CoreFoundation)
    find_library(CARBON_LIBRARY Carbon)
    
    # force cpp -> Objective-C++
    set_source_files_properties(
        src/5dc_ui_app.cpp
        PROPERTIES
        COMPILE_FLAGS "-x objective-c++"
    )

    target_link_libraries(5DCUI PRIVATE
        ${COCOA_LIBRARY}
        ${IOKIT_LIBRARY}
        ${COREFOUNDATION_LIBRARY}
        ${CARBON_LIBRARY} # link Carbon
        "-framework OpenGL"
        "-framework Foundation"
    )
    
elseif(UNIX AND NOT APPLE)
    # Linux dependencies
    find_package(PkgConfig REQUIRED)
    pkg_check_modules(GTK3 REQUIRED gtk+-3.0)
    
    find_package(X11 REQUIRED COMPONENTS Xi Xcursor Xrandr Xinerama)
    find_package(OpenGL REQUIRED)
    
    target_include_directories(5DCUI PRIVATE
        ${GTK3_INCLUDE_DIRS}
    )
    
    target_link_libraries(5DCUI PRIVATE
        ${X11_LIBRARIES}
        ${X11_Xi_LIB}
        ${X11_Xcursor_LIB}
        ${X11_Xrandr_LIB}
        ${X11_Xinerama_LIB}
        ${OPENGL_LIBRARIES}
        ${GTK3_LIBRARIES}
        pthread
        dl
        udev
    )
endif()

# Set output directory
set_target_properties(5DCUI PROPERTIES
    RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin
)