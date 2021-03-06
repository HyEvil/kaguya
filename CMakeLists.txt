cmake_minimum_required(VERSION 2.6)
project(Kaguya)

include(cmake/FindLua.cmake)

include_directories(${LUA_INCLUDE_DIRS})
link_directories(${LUA_LIBRARY_DIRS})
include_directories("include")

file(GLOB KAGUYA_HEADER RELATIVE ${CMAKE_CURRENT_SOURCE_DIR}
  include/kaguya/*.hpp
  include/kaguya/detail/*.hpp)

include_directories("./")

if(ADDITIONAL_INCLUDE_PATH)
include_directories(${ADDITIONAL_INCLUDE_PATH})
endif()

if(NOT MSVC)#-Wall nonsense on MSVC
add_definitions("-Wall")
#add_definitions("-std=c++11")
endif(NOT MSVC)

if(KAGUYA_NO_USERDATA_TYPE_CHECK)
  add_definitions("-DKAGUYA_NO_USERDATA_TYPE_CHECK=1")
endif()

#check for address sanitizer support
if(${CMAKE_CXX_COMPILER_ID} STREQUAL "GNU")
    if (${CMAKE_CXX_COMPILER_VERSION} VERSION_GREATER 4.8)
      set(HAVE_FLAG_SANITIZE_ADDRESS TRUE)
    endif()
elseif (${CMAKE_CXX_COMPILER_ID} STREQUAL "Clang")
    if (${CMAKE_CXX_COMPILER_VERSION} VERSION_GREATER 3.2)
      set(HAVE_FLAG_SANITIZE_ADDRESS TRUE)
    endif()
    #detect stack-buffer-overflow at clang++3.4(on ubuntu 12.04) with luajit but don't know reason
    if(${CMAKE_CXX_COMPILER_VERSION} VERSION_EQUAL 3.4)
      if(${LUA_SEARCH_LIB_NAME} STREQUAL "luajit")
        unset(HAVE_FLAG_SANITIZE_ADDRESS)
      endif()
    endif()
endif()
if(CYGWIN OR WIN32)
      unset(HAVE_FLAG_SANITIZE_ADDRESS)
endif()

set(BENCHMARK_SRCS benchmark/benchmark.cpp benchmark/benchmark_function.cpp benchmark/benchmark_function.hpp)
add_executable(benchmark ${BENCHMARK_SRCS} ${KAGUYA_HEADER})
target_link_libraries(benchmark ${LUA_LIBRARIES})


enable_testing()
if(COVERAGE)
SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -O0 -Wall -Woverloaded-virtual -Wwrite-strings -fprofile-arcs -ftest-coverage -coverage -fno-inline -fno-inline-small-functions -fno-default-inline")
SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -g -O0 -Wall -W -fprofile-arcs -ftest-coverage -coverage -fno-inline -fno-inline-small-functions -fno-default-inline")
SET(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fprofile-arcs -ftest-coverage -coverage -fno-inline -fno-inline-small-functions -fno-default-inline")
endif(COVERAGE)

file(GLOB TEST_SRCS files RELATIVE ${CMAKE_SOURCE_DIR} test/test_*.cpp)

add_executable(test_runner ${TEST_SRCS}  ${KAGUYA_HEADER})
target_link_libraries(test_runner ${LUA_LIBRARIES})
if(HAVE_FLAG_SANITIZE_ADDRESS AND NOT DISABLE_ADDRESS_SANITIZER)
SET_TARGET_PROPERTIES(test_runner PROPERTIES COMPILE_FLAGS "-fsanitize=address -fno-omit-frame-pointer")
SET_TARGET_PROPERTIES(test_runner PROPERTIES LINK_FLAGS "-fsanitize=address")
endif(HAVE_FLAG_SANITIZE_ADDRESS AND NOT DISABLE_ADDRESS_SANITIZER)
add_test(
  NAME test_runner
  COMMAND $<TARGET_FILE:test_runner>
  WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}/test)
