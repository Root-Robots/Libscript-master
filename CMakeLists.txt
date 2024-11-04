cmake_minimum_required(VERSION 3.9)
project(libscript)

set (CMAKE_CXX_STANDARD 14)

if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "MSVC")
  # disables warning about DLL export 
  # this is annoying when using C++ STL
  add_compile_options(/wd4251)  
  add_compile_options(/wd4275) 
endif()

##################################################################
###### coverage build
##################################################################

if(ENABLE_COVERAGE)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g ")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O0")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fprofile-arcs")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -ftest-coverage")
  set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} --coverage")
endif()

##################################################################
###### libscript
##################################################################

file(GLOB_RECURSE SRC_FILES ${CMAKE_CURRENT_SOURCE_DIR}/src/*.cpp)
file(GLOB_RECURSE HDR_FILES 
  ${CMAKE_CURRENT_SOURCE_DIR}/include/*.h
  ${CMAKE_CURRENT_SOURCE_DIR}/src/*.h
)
add_library(libscript SHARED ${HDR_FILES} ${SRC_FILES})
target_include_directories(libscript PUBLIC "${CMAKE_CURRENT_SOURCE_DIR}/include")
# the following line causes trouble with standard headers like string.h... :(
#target_include_directories(libscript PRIVATE "${CMAKE_CURRENT_SOURCE_DIR}/include/script")

foreach(_source IN ITEMS ${HDR_FILES} ${SRC_FILES})
    get_filename_component(_source_path "${_source}" PATH)
    file(RELATIVE_PATH _source_path_rel "${CMAKE_CURRENT_SOURCE_DIR}" "${_source_path}")
    string(REPLACE "/" "\\" _group_path "${_source_path_rel}")
    source_group("${_group_path}" FILES "${_source}")
endforeach()

target_compile_definitions(libscript PRIVATE -DLIBSCRIPT_COMPILE_LIBRARY)

##################################################################
###### tests
##################################################################

set(LIBSCRIPT_PROJECT_DIR ${CMAKE_CURRENT_SOURCE_DIR})

set(LIBSCRIPT_BUILD_TESTS ON CACHE BOOL "whether to build the tests")

if (LIBSCRIPT_BUILD_TESTS)
  add_subdirectory(tests)
endif()

set(LIBSCRIPT_BUILD_EXAMPLES ON CACHE BOOL "whether to build the examples")

if (LIBSCRIPT_BUILD_EXAMPLES)
  add_subdirectory(examples)
endif()
