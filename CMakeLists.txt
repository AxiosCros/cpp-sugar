cmake_minimum_required(VERSION 3.7 FATAL_ERROR)

project(csugar VERSION "0.1.0")
option(ENABLE_UNIT_TESTS "Enable unit tests" Off)

message("<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<< General set up >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>")
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
message(STATUS "C++ compiler flags: ${CMAKE_CXX_FLAGS}")

if (CMAKE_CXX_COMPILER_LOADED)
    message(STATUS "The C++ compiler ID is: ${CMAKE_CXX_COMPILER_ID}")
    message(STATUS "The C++ compiler version is: ${CMAKE_CXX_COMPILER_VERSION}")
endif ()

if (NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
endif ()

message(STATUS "Build type: ${CMAKE_BUILD_TYPE}")

message("<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<< Target set up >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>")
message(STATUS "Enable testing: ${ENABLE_UNIT_TESTS}")
message(STATUS "Enable coverage: ${ENABLE_COVERAGE}")
message(STATUS "Target Name : ${PROJECT_NAME}")
add_library(${PROJECT_NAME} SHARED "")
set(headers
        include/csugar.h)
target_sources(${PROJECT_NAME}
        PRIVATE
        src/string.cpp
        src/json.cpp
        src/datetime.cpp
        src/crypt.cpp
        ${headers})
target_include_directories(${PROJECT_NAME}
        PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}/include)
set_target_properties(${PROJECT_NAME}
        PROPERTIES
        POSITION_INDEPENDENT_CODE 1
        SOVERSION ${PROJECT_VERSION_MAJOR}
        OUTPUT_NAME "csugar"
        DEBUG_POSTFIX "_d"
        PUBLIC_HEADER "${headers}"
        MACOSX_RPATH ON
        WINDOWS_EXPORT_ALL_SYMBOLS ON)
if (ENABLE_UNIT_TESTS)
    OPTION(ENABLE_COVERAGE "Use gcov" OFF)
    if (ENABLE_COVERAGE AND CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
        set(CMAKE_BUILD_TYPE "Coverage")
        set(CMAKE_MODULE_PATH ${PROJECT_SOURCE_DIR}/cmake)
        include(CodeCoverage)
        setup_target_for_coverage(${PROJECT_NAME}_coverage ${PROJECT_NAME}_test coverage)
    endif ()
    include(CTest)
    enable_testing()
    add_executable(tests_csugar
            tests/main.cpp
            tests/crypt.cpp
            tests/datetime.cpp
            tests/json.cpp
            tests/string.cpp)
endif ()

message("<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<< External set up >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>")
find_package(OpenSSL 1.0.0 REQUIRED)
add_subdirectory(external)
target_link_libraries(${PROJECT_NAME}
        PRIVATE
        jsoncpp
        crypto
        ${Boost_LIBRARIES})
if (ENABLE_UNIT_TESTS)
    target_include_directories(tests_csugar
            PRIVATE
            ${CMAKE_CURRENT_SOURCE_DIR}/include)
    target_link_libraries(tests_csugar
            PRIVATE
            ${PROJECT_NAME}
            gtest_main
            gmock_main
            crypto
            jsoncpp
            ${Boost_LIBRARIES})
    add_test(NAME google_test
            COMMAND $<TARGET_FILE:tests_csugar>)
endif ()

message("<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<<< Install set up >>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>>")
message(STATUS "Project will be installed to ${CMAKE_INSTALL_PREFIX}")

include(GNUInstallDirs)

set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/${CMAKE_INSTALL_LIBDIR})
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/${CMAKE_INSTALL_LIBDIR})
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJECT_BINARY_DIR}/${CMAKE_INSTALL_BINDIR})

set(INSTALL_LIBDIR ${CMAKE_INSTALL_LIBDIR} CACHE PATH "Installation directory for libraries")
set(INSTALL_BINDIR ${CMAKE_INSTALL_BINDIR} CACHE PATH "Installation directory for executables")
set(INSTALL_INCLUDEDIR ${CMAKE_INSTALL_INCLUDEDIR} CACHE PATH "Installation directory for header files")
if (WIN32 AND NOT CYGWIN)
    set(DEF_INSTALL_CMAKEDIR CMake)
else ()
    set(DEF_INSTALL_CMAKEDIR share/cmake/${PROJECT_NAME})
endif ()
set(INSTALL_CMAKEDIR ${DEF_INSTALL_CMAKEDIR} CACHE PATH "Installation directory for CMake files")

foreach (p LIB BIN INCLUDE CMAKE)
    file(TO_NATIVE_PATH ${CMAKE_INSTALL_PREFIX}/${INSTALL_${p}DIR} _path)
    message(STATUS "Installing ${p} components to ${_path}")
    unset(_path)
endforeach ()

# Prepare RPATH
file(RELATIVE_PATH _rel ${CMAKE_INSTALL_PREFIX}/${INSTALL_BINDIR} ${CMAKE_INSTALL_PREFIX})
if (APPLE)
    set(_rpath "@loader_path/${_rel}")
else ()
    set(_rpath "\$ORIGIN/${_rel}")
endif ()
file(TO_NATIVE_PATH "${_rpath}/${INSTALL_LIBDIR}" install_RPATH)

set_target_properties(${PROJECT_NAME}
        PROPERTIES
        MACOSX_RPATH ON
        SKIP_BUILD_RPATH OFF
        BUILD_WITH_INSTALL_RPATH OFF
        INSTALL_RPATH "${install_RPATH}"
        INSTALL_RPATH_USE_LINK_PATH ON)

install(
        TARGETS
        ${PROJECT_NAME}
        EXPORT
        csugarTargets
        ARCHIVE
        DESTINATION ${INSTALL_LIBDIR}
        COMPONENT lib
        RUNTIME
        DESTINATION ${INSTALL_BINDIR}
        COMPONENT bin
        LIBRARY
        DESTINATION ${INSTALL_LIBDIR}
        COMPONENT lib
        PUBLIC_HEADER
        DESTINATION ${INSTALL_INCLUDEDIR}/
        COMPONENT dev
)

# install cmake config
install(
        EXPORT
        csugarTargets
        NAMESPACE
        "csugar::"
        DESTINATION
        ${INSTALL_CMAKEDIR}
        COMPONENT
        dev
)
include(CMakePackageConfigHelpers)

write_basic_package_version_file(
        ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake
        VERSION ${PROJECT_VERSION}
        COMPATIBILITY SameMajorVersion
)

configure_package_config_file(
        ${PROJECT_SOURCE_DIR}/cmake/${PROJECT_NAME}Config.cmake.in
        ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake
        INSTALL_DESTINATION ${INSTALL_CMAKEDIR}
)

install(
        FILES
        ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}Config.cmake
        ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake
        DESTINATION
        ${INSTALL_CMAKEDIR}
)
