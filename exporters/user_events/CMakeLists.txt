cmake_minimum_required(VERSION 3.12)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

if(WIN32)
  message(FATAL_ERROR "user_events is Linux only for now")
endif()

option(BUILD_EXAMPLE "Build example" ON)
option(BUILD_TESTING "Build tests" ON)

set(MAIN_PROJECT OFF)

if(CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_SOURCE_DIR)
  project(opentelemetry-userevents-logs)
  set(MAIN_PROJECT ON)
endif()

add_definitions(-DHAVE_CONSOLE_LOG -DENABLE_LOGS_PREVIEW)

if(MAIN_PROJECT)
  # TODO: cleanup the dependent packages
  find_package(Protobuf REQUIRED)
  find_package(CURL REQUIRED)
  find_package(nlohmann_json REQUIRED)
  find_package(opentelemetry-cpp REQUIRED)
endif()

add_subdirectory(third_party/LinuxTracepoints)

include_directories(include)

add_library(opentelemetry_exporter_user_events_logs
            src/logs_exporter.cc src/recordable.cc src/utils.cc)

if(MAIN_PROJECT)
  target_include_directories(opentelemetry_exporter_user_events_logs
                             PRIVATE ${OPENTELEMETRY_CPP_INCLUDE_DIRS})
  target_link_libraries(opentelemetry_exporter_user_events_logs
                        PUBLIC ${OPENTELEMETRY_CPP_LIBRARIES})
else()
  target_link_libraries(
    opentelemetry_exporter_user_events_logs
    PUBLIC opentelemetry_logs opentelemetry_resources opentelemetry_common)
endif()

target_link_libraries(opentelemetry_exporter_user_events_logs
                      PUBLIC eventheader-tracepoint tracepoint)

set_target_properties(opentelemetry_exporter_user_events_logs
                      PROPERTIES EXPORT_NAME logs)

if(BUILD_EXAMPLE)

  add_executable(user_events_logs example/main.cc example/foo_library.cc)
  target_link_libraries(
    user_events_logs ${CMAKE_THREAD_LIBS_INIT} opentelemetry_logs
    opentelemetry_exporter_user_events_logs)

endif(BUILD_EXAMPLE)

if(BUILD_TESTING)
  if(EXISTS ${CMAKE_BINARY_DIR}/lib/libgtest.a)
    # Prefer GTest from build tree. GTest is not always working with
    # CMAKE_PREFIX_PATH
    set(GTEST_INCLUDE_DIRS
        ${CMAKE_CURRENT_SOURCE_DIR}/third_party/googletest/googletest/include
        ${CMAKE_CURRENT_SOURCE_DIR}/third_party/googletest/googlemock/include)
    set(GTEST_BOTH_LIBRARIES
        ${CMAKE_BINARY_DIR}/lib/libgtest.a
        ${CMAKE_BINARY_DIR}/lib/libgtest_main.a
        ${CMAKE_BINARY_DIR}/lib/libgmock.a)
  else()
    find_package(GTest REQUIRED)
  endif()
  include_directories(SYSTEM ${GTEST_INCLUDE_DIRS})
  include_directories(SYSTEM test/decoder)
  enable_testing()
  include(GoogleTest)
  # build tests for user_events logs
  add_executable(userevents_logs_exporter_test test/logs_exporter_test.cc)
  target_link_libraries(
    userevents_logs_exporter_test ${GTEST_BOTH_LIBRARIES}
    ${CMAKE_THREAD_LIBS_INIT} opentelemetry_exporter_user_events_logs)

  gtest_add_tests(
    TARGET userevents_logs_exporter_test
    TEST_PREFIX exporter.
    TEST_LIST userevents_logs_exporter_test)
endif()

find_package(benchmark CONFIG REQUIRED)
add_executable(user_events_logger_benchmark benchmark/logger_benchmark.cc)
target_link_libraries(
  user_events_logger_benchmark benchmark::benchmark ${CMAKE_THREAD_LIBS_INIT}
  ${CMAKE_THREAD_LIBS_INIT} opentelemetry_logs
  opentelemetry_exporter_user_events_logs)

install(
  TARGETS opentelemetry_exporter_user_events_logs
  EXPORT "${PROJECT_NAME}-target"
  RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR})

install(
  DIRECTORY include/opentelemetry/exporters/userevents
  DESTINATION include/opentelemetry/exporters
  FILES_MATCHING
  PATTERN "*.h")
