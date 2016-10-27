cmake_minimum_required(VERSION 3.5)

project(composition)

if(NOT WIN32)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11 -Wall -Wextra")
endif()

find_package(ament_cmake REQUIRED)
find_package(class_loader REQUIRED)
find_package(rclcpp REQUIRED)
find_package(rmw_implementation_cmake REQUIRED)
find_package(std_msgs REQUIRED)

include_directories(include)

macro(targets)
  get_rclcpp_information("${rmw_implementation}" "rclcpp${target_suffix}")

  add_library(talker_component${target_suffix} SHARED
    src/talker_component.cpp
  )
  target_compile_definitions(talker_component${target_suffix}
    PRIVATE "COMPOSITION_BUILDING_DLL")
  ament_target_dependencies(talker_component${target_suffix}
    "rclcpp${target_suffix}"
    "std_msgs"
    "class_loader")

  add_library(listener_component${target_suffix} SHARED
    src/listener_component.cpp
  )
  target_compile_definitions(listener_component${target_suffix}
    PRIVATE "COMPOSITION_BUILDING_DLL")
  ament_target_dependencies(listener_component${target_suffix}
    "rclcpp${target_suffix}"
    "std_msgs"
    "class_loader")

  add_executable(manual_composition${target_suffix}
    src/manual_composition.cpp
  )
  target_link_libraries(manual_composition${target_suffix}
    talker_component${target_suffix}
    listener_component${target_suffix})
  ament_target_dependencies(manual_composition${target_suffix}
    "rclcpp${target_suffix}")

  add_executable(linktime_composition${target_suffix}
    src/linktime_composition.cpp
  )
  set(libs
    talker_component${target_suffix}
    listener_component${target_suffix})
  if(CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    set(libs
      "-Wl,--no-as-needed"
      ${libs}
      "-Wl,--as-needed")
  endif()
  target_link_libraries(linktime_composition${target_suffix} ${libs})
  ament_target_dependencies(linktime_composition${target_suffix}
    "rclcpp${target_suffix}"
    "class_loader")

  add_executable(dlopen_composition${target_suffix}
    src/dlopen_composition.cpp
  )
  ament_target_dependencies(dlopen_composition${target_suffix}
    "rclcpp${target_suffix}"
    "class_loader")

  install(TARGETS
    talker_component${target_suffix}
    listener_component${target_suffix}
    manual_composition${target_suffix}
    linktime_composition${target_suffix}
    dlopen_composition${target_suffix}
    ARCHIVE DESTINATION lib
    LIBRARY DESTINATION lib
    RUNTIME DESTINATION bin)
endmacro()

call_for_each_rmw_implementation(targets GENERATE_DEFAULT)

if(BUILD_TESTING)
  find_package(ament_lint_auto REQUIRED)
  ament_lint_auto_find_test_dependencies()

  macro(tests)
    set(MANUAL_COMPOSITION_EXECUTABLE $<TARGET_FILE:manual_composition${target_suffix}>)
    set(LINKTIME_COMPOSITION_EXECUTABLE $<TARGET_FILE:linktime_composition${target_suffix}>)
    set(DLOPEN_COMPOSITION_EXECUTABLE $<TARGET_FILE:dlopen_composition${target_suffix}>)
    set(TALKER_LIBRARY $<TARGET_FILE:talker_component${target_suffix}>)
    set(LISTENER_LIBRARY $<TARGET_FILE:listener_component${target_suffix}>)
    set(EXPECTED_OUTPUT "${CMAKE_CURRENT_SOURCE_DIR}/test/composition")

    configure_file(
      test/test_composition.py.in
      test_composition${target_suffix}.py.genexp
      @ONLY
    )
    file(GENERATE
      OUTPUT test_composition${target_suffix}_$<CONFIGURATION>.py
      INPUT test_composition${target_suffix}.py.genexp)
    ament_add_nose_test(test_composition${target_suffix}
      "${CMAKE_CURRENT_BINARY_DIR}/test_composition${target_suffix}_$<CONFIGURATION>.py"
      TIMEOUT 30)
  endmacro()

  call_for_each_rmw_implementation(tests)
endif()

ament_package()
