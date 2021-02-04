
cmake_minimum_required(VERSION 3.10)
project(glsl-optimizer)

if(CMAKE_SIZEOF_VOID_P EQUAL 8)
    set(ARCH_POSTFIX 64)
else()
    set(ARCH_POSTFIX "")
endif()

set(CMAKE_DEBUG_POSTFIX d)

message("Configuration: ARCH_POSTFIX:${ARCH_POSTFIX} CMAKE_DEBUG_POSTFIX:${CMAKE_DEBUG_POSTFIX}")

set(OpenGL_GL_PREFERENCE GLVND)
find_package(OpenGL REQUIRED)

include_directories(include)
include_directories(src/mesa)
include_directories(src/mapi)
include_directories(src/glsl)
include_directories(src)

if(WIN32)
	include_directories(src/getopt)
endif()

option (DEBUG "Enable debugging" FALSE)

if(${DEBUG} MATCHES "on")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -O0")
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -g -O0")
else()
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Os -DNDEBUG")
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Os -DNDEBUG")
	set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -s")
endif()

file(GLOB glcpp-library_sources src/glsl/glcpp/*.c src/util/*.c)
# file(GLOB glcpp-library_sources_remove src/glsl/glcpp/glcpp.c)
# list(REMOVE_ITEM glcpp-library_sources ${glcpp-library_sources_remove})
add_library(glcpp-library${ARCH_POSTFIX} ${glcpp-library_sources})

file(GLOB mesa_sources src/mesa/program/*.c src/mesa/main/*.c)
add_library(mesa${ARCH_POSTFIX} ${mesa_sources})

file(GLOB glsl_sources src/glsl/*.cpp src/glsl/*.c)
file(GLOB glsl_sources_remove src/glsl/main.cpp src/glsl/builtin_stubs.cpp)
list(REMOVE_ITEM glsl_sources ${glsl_sources_remove})

add_library(glsl_optimizer${ARCH_POSTFIX} ${glsl_sources})
target_link_libraries(glsl_optimizer${ARCH_POSTFIX} glcpp-library${ARCH_POSTFIX} mesa${ARCH_POSTFIX})

# Single lib combining all three of the above (for linux to workaround linking
# problems but could become default for all).
add_library(glsl_optimizer_combined${ARCH_POSTFIX} ${glcpp-library_sources} ${mesa_sources} ${glsl_sources})

if(WIN32)
	add_executable(glsl_compiler src/glsl/main.cpp src/getopt/getopt_long.c)
else()
	add_executable(glsl_compiler src/glsl/main.cpp)
endif()
target_link_libraries(glsl_compiler PRIVATE glsl_optimizer${ARCH_POSTFIX})

file(GLOB glsl_test_sources tests/*.cpp)
add_executable(glsl_test ${glsl_test_sources})
target_link_libraries(glsl_test PRIVATE glsl_optimizer${ARCH_POSTFIX})
target_link_libraries(glsl_test PRIVATE OpenGL::GL)

file(GLOB glslopt_sources contrib/glslopt/*.cpp)
add_executable(glslopt ${glslopt_sources})
target_link_libraries(glslopt PRIVATE glsl_optimizer${ARCH_POSTFIX})

#add_executable(glcpp src/glsl/glcpp/glcpp.c)
#target_link_libraries(glcpp PRIVATE glsl_optimizer${ARCH_POSTFIX})
