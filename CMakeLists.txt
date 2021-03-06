cmake_minimum_required(VERSION 2.8)
set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/CMake;${CMAKE_MODULE_PATH}")
include(Utilities)
project(pic_project)

# User Flags
option(BUILD_DEPENDENCIES "Whether or not to build depending libraries from source" ON)

# Developer Flags
option(BUILD_TEST "Build the testing tree." OFF)
option(CODE_COVERAGE "Enable coverage reporting" OFF)
option(COMPILER_WARNINGS "Enable all compiler warnings." OFF)
option(ADDRESS_SANITIZER "Build with AddressSanitizer." OFF)
option(MEMORY_SANITIZER "Build with MemorySanitizer." OFF)
option(THREAD_SANITIZER "Build with ThreadSanitizer." OFF)
option(UNDEFINED_BEHAVIOR_SANITIZER "Build with UndefinedBehaviorSanitizer." OFF)
option(BUILD_DEBUG_HEAP "Build debug heap with guard bands and validation." OFF)
option(ALIGNED_MEMORY_MODEL "Aligned memory model ONLY." OFF)

if(CODE_COVERAGE)
  set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -O0 -g -fprofile-arcs -ftest-coverage")
  set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} --coverage")
endif()

function(enableSanitizer SANITIZER)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O0 -g -fsanitize=${SANITIZER} -fno-omit-frame-pointer" PARENT_SCOPE)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -O0 -g -fsanitize=${SANITIZER} -fno-omit-frame-pointer -fno-optimize-sibling-calls" PARENT_SCOPE)
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fsanitize=${SANITIZER}" PARENT_SCOPE)
endfunction()

if(ADDRESS_SANITIZER)
  enableSanitizer("address")
endif()
if(MEMORY_SANITIZER)
  enableSanitizer("memory")
endif()
if(THREAD_SANITIZER)
  enableSanitizer("thread")
endif()
if(UNDEFINED_BEHAVIOR_SANITIZER)
  enableSanitizer("undefined")
endif()
if(BUILD_DEBUG_HEAP)
  add_definitions(-DHEAP_DEBUG)
endif()
if(ALIGNED_MEMORY_MODEL)
  add_definitions(-DALIGNED_MEMORY_MODEL)
endif()

if(NOT KINESIS_VIDEO_OPEN_SOURCE_SRC)
  set(KINESIS_VIDEO_OPEN_SOURCE_SRC ${CMAKE_SOURCE_DIR}/open-source/local)
endif()

if(BUILD_DEPENDENCIES)
  if(NOT EXISTS ${KINESIS_VIDEO_OPEN_SOURCE_SRC})
    file(MAKE_DIRECTORY ${KINESIS_VIDEO_OPEN_SOURCE_SRC}/local)
  endif()

  if (NOT OPEN_SRC_INSTALL_PREFIX)
    set(OPEN_SRC_INSTALL_PREFIX ${KINESIS_VIDEO_OPEN_SOURCE_SRC}/local)
    set(ENV{PKG_CONFIG_PATH}
          "$ENV{PKG_CONFIG_PATH}:${OPEN_SRC_INSTALL_PREFIX}/lib/pkgconfig")
    set(CMAKE_PREFIX_PATH ${CMAKE_PREFIX_PATH} ${OPEN_SRC_INSTALL_PREFIX})
  endif()

  if(BUILD_TEST)
    build_dependency(gtest)
  endif()
endif()

set(KINESIS_VIDEO_PIC_SRC ${CMAKE_CURRENT_SOURCE_DIR})

set(THREADS_PREFER_PTHREAD_FLAG ON)
find_package(Threads)

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fPIC")

file(GLOB PIC_CLIENT_SOURCE_FILES "src/client/src/*.c")

file(GLOB PIC_DURATION_SOURCE_FILES "src/duration/src/*.c")

file(GLOB PIC_HEAP_SOURCE_FILES "src/heap/src/*.c")

file(GLOB PIC_MKVGEN_SOURCE_FILES "src/mkvgen/src/*.c")

file(GLOB PIC_STATE_SOURCE_FILES "src/state/src/*.c")

file(GLOB PIC_TRACE_SOURCE_FILES "src/trace/src/*.c")

file(GLOB PIC_UTILS_SOURCE_FILES "src/utils/src/*.c")

file(GLOB PIC_VIEW_SOURCE_FILES "src/view/src/*.c")

file(GLOB PIC_TEST_SOURCE_FILES "src/*/tst/*.cpp")

file(GLOB PIC_HEADERS "${KINESIS_VIDEO_PIC_SRC}/src/*/include")

include_directories(${PIC_HEADERS})
include_directories(${KINESIS_VIDEO_PIC_SRC})

add_library(
  kvspic STATIC
  ${PIC_CLIENT_SOURCE_FILES}
  ${PIC_DURATION_SOURCE_FILES}
  ${PIC_HEAP_SOURCE_FILES}
  ${PIC_MKVGEN_SOURCE_FILES}
  ${PIC_STATE_SOURCE_FILES}
  ${PIC_TRACE_SOURCE_FILES}
  ${PIC_UTILS_SOURCE_FILES}
  ${PIC_VIEW_SOURCE_FILES})
target_link_libraries(kvspic ${CMAKE_DL_LIBS} Threads::Threads)
if(UNIX AND NOT APPLE)
  # rt needed for clock_gettime
  target_link_libraries(kvspic rt)
endif()

add_library(kvspicClient STATIC ${PIC_CLIENT_SOURCE_FILES})
target_link_libraries(kvspicClient)

add_library(kvspicState STATIC ${PIC_STATE_SOURCE_FILES})
target_link_libraries(kvspicState)

add_library(kvspicUtils STATIC ${PIC_UTILS_SOURCE_FILES})
target_link_libraries(kvspicUtils ${CMAKE_DL_LIBS} Threads::Threads)
if(UNIX AND NOT APPLE)
  # rt needed for clock_gettime
  target_link_libraries(kvspicUtils rt)
endif()

if(BUILD_TEST)
  set(CMAKE_CXX_STANDARD 11)

  set(GTEST_ROOT ${OPEN_SRC_INSTALL_PREFIX})
  find_package(GTest REQUIRED)

  add_executable(kvspic_test ${PIC_TEST_SOURCE_FILES}
          ${PIC_CLIENT_SOURCE_FILES}
          ${PIC_DURATION_SOURCE_FILES}
          ${PIC_HEAP_SOURCE_FILES}
          ${PIC_MKVGEN_SOURCE_FILES}
          ${PIC_STATE_SOURCE_FILES}
          ${PIC_TRACE_SOURCE_FILES}
          ${PIC_UTILS_SOURCE_FILES}
          ${PIC_VIEW_SOURCE_FILES})
  target_compile_definitions(kvspic_test PRIVATE ALIGNED_MEMORY_MODEL=TRUE)
  target_link_libraries(kvspic_test GTest::GTest GTest::Main ${CMAKE_DL_LIBS} Threads::Threads)
  if(UNIX AND NOT APPLE)
    # rt needed for clock_gettime
    target_link_libraries(kvspic_test rt)
  endif()
endif()
