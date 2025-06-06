# libjsonnet++

set(LIBJSONNETPP_HEADERS
    ../include/libjsonnet++.h
)

set(LIBJSONNETPP_SOURCE
    libjsonnet++.cpp
)

add_library(libjsonnet++ SHARED ${LIBJSONNETPP_HEADERS} ${LIBJSONNETPP_SOURCE})
add_dependencies(libjsonnet++ jsonnet)
target_link_libraries(libjsonnet++ libjsonnet)

# Version extraction logic duplicated from ../core/CMakeLists.txt
# TODO: Put this logic somewhere else.
file(STRINGS ${CMAKE_CURRENT_SOURCE_DIR}/../include/libjsonnet.h JSONNET_VERSION_DEF
     REGEX "[#]define[ \t]+LIB_JSONNET_VERSION[ \t]+")
string(REGEX REPLACE ".*\"v([^\"]+)\".*" "\\1" JSONNET_VERSION ${JSONNET_VERSION_DEF})
message("Extracted Jsonnet version: " ${JSONNET_VERSION})

# CMake prepends CMAKE_SHARED_LIBRARY_PREFIX to shared libraries, so without
# this step the output would be |liblibjsonnet|.
set_target_properties(libjsonnet++ PROPERTIES OUTPUT_NAME jsonnet++
	VERSION     "${JSONNET_VERSION}"
	SOVERSION   "0"
	PUBLIC_HEADER "${LIB_HEADER}")
install(TARGETS libjsonnet++
	RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}"
	LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}"
	ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
	PUBLIC_HEADER DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}")
target_include_directories(libjsonnet++ INTERFACE
  $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/../include>)

if (BUILD_STATIC_LIBS)
    # Static library for jsonnet command-line tool.
    add_library(libjsonnet++_static STATIC ${LIBJSONNETPP_SOURCE})
    add_dependencies(libjsonnet++_static jsonnet)
    target_link_libraries(libjsonnet++_static libjsonnet_static)
    set_target_properties(libjsonnet++_static PROPERTIES OUTPUT_NAME jsonnet++)
    install(TARGETS libjsonnet++_static DESTINATION "${CMAKE_INSTALL_LIBDIR}")
    target_include_directories(libjsonnet++_static INTERFACE
      $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/../include>)
endif()

if (BUILD_SHARED_BINARIES OR NOT BUILD_STATIC_LIBS)
    add_library(libjsonnet++_for_binaries ALIAS libjsonnet++)
else()
    add_library(libjsonnet++_for_binaries ALIAS libjsonnet++_static)
endif()

# Tests
function(add_test_executablepp test_name)
    if (EXISTS ${CMAKE_CURRENT_LIST_DIR}/${test_name}.cpp)
        set(TEST_EXT cpp)
    else()
        set(TEST_EXT c)
    endif()
    add_executable(${test_name} ${test_name}.${TEST_EXT})

    add_dependencies(${test_name} libjsonnet++_for_binaries libjsonnet_for_binaries GTest::Main)
    target_link_libraries(
        ${test_name} GTest::GTest GTest::Main libjsonnet++_for_binaries libjsonnet_for_binaries)
endfunction()

if (BUILD_TESTS)
    add_test_executablepp(libjsonnet++_test)
    # Run this in the source tree because it needs to access testdata files.
    add_test(NAME libjsonnet++_test COMMAND ${GLOBAL_OUTPUT_PATH}/libjsonnet++_test WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/..")

    add_test_executablepp(libjsonnet_test_locale)
    add_test(NAME libjsonnet_test_locale COMMAND ${GLOBAL_OUTPUT_PATH}/libjsonnet_test_locale WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/..")
endif()
