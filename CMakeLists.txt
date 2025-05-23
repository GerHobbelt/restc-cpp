cmake_minimum_required(VERSION 3.10)

if (DEFINED ENV{RESTC_CPP_VERSION})
    set(RESTC_CPP_VERSION $ENV{RESTC_CPP_VERSION})
endif()

if (NOT DEFINED RESTC_CPP_VERSION)
    set(RESTC_CPP_VERSION 1.0.0)
endif()

if(NOT DEFINED RESTC_BOOST_VERSION)
    set(RESTC_BOOST_VERSION 1.58)
endif()

project (restc-cpp VERSION ${RESTC_CPP_VERSION})

message(STATUS "Building restc-cpp version ${PROJECT_VERSION}")

include(CheckCXXCompilerFlag)

if (MSVC)
    # Thank you Microsoft. Its so nice of you to give us all these meaningful reasons to stay up all night.
    check_cxx_compiler_flag("/std:c++20" COMPILER_SUPPORTS_CXX20)
    check_cxx_compiler_flag("/std:c++17" COMPILER_SUPPORTS_CXX17)
    add_compile_options(/Zc:__cplusplus)
else()
    check_cxx_compiler_flag("-std=c++20" COMPILER_SUPPORTS_CXX20)
    check_cxx_compiler_flag("-std=c++17" COMPILER_SUPPORTS_CXX17)
endif()

if (NOT DEFINED INSTALL_RAPIDJSON_HEADERS)
    option(INSTALL_RAPIDJSON_HEADERS "Install rapidjson headers when make install is executed" ON)
endif()

option(RESTC_CPP_WITH_UNIT_TESTS "Enable Unit Testing" ON)

option(RESTC_CPP_AUTORUN_UNIT_TESTS "Run Unit Tests automatically after build" OFF)

option(RESTC_CPP_WITH_FUNCTIONALT_TESTS "Enable Functional Testing" ON)

option(RESTC_USE_LEGACY_BOOST_FIND "Use the old Boost find module" OFF)

set(GTEST_TAG "main" CACHE STRING "Gtest branch to use. Required on older Linux versions because newer gtest requure newer cmake!")
set(LOGFAULT_TAG "master" CACHE STRING "Logfault branch to use. Required on older Linux versions because newer gtest requure newer cmake!")

find_package(RapidJSON QUIET)

if (NOT RapidJSON_FOUND AND INSTALL_RAPIDJSON_HEADERS)
    message(STATUS "Rapidjson not found. Adding it as an external project.")
    set(RESTC_CPP_EXTERNAL_DEPS ${RESTC_CPP_EXTERNAL_DEPS} externalRapidJson)
    set(restc_cpp_add_rapidjson ON)
endif()

if (RESTC_CPP_LOG_WITH_LOGFAULT)
    find_path(LOGFAULT logfault/logfault.h)
    if (LOGFAULT)
        message ("Using existing logfault")
    else()
        message ("Embedding logfault header only library")
        set(RESTC_CPP_EXTERNAL_DEPS ${RESTC_CPP_EXTERNAL_DEPS} externalLogfault)
        set(restc_cpp_add_logfault ON)
    endif()
endif()

include(cmake_scripts/external-projects.cmake)

if (EXISTS ${CMAKE_BINARY_DIR}/conanbuildinfo.cmake)
    message(STATUS "Using conan configuration: ${CMAKE_BINARY_DIR}/conanbuildinfo.cmake")
    include(${CMAKE_BINARY_DIR}/conanbuildinfo.cmake)
    conan_basic_setup()
endif()

if (NOT DEFINED RESTC_ROOT_DIR)
    set(RESTC_ROOT_DIR ${CMAKE_CURRENT_SOURCE_DIR})
endif()

option(WITH_APIDOC "Generate Doxygen documentation")

option(RESTC_CPP_WITH_EXAMPLES "Compile examples" ON)

option(RESTC_CPP_WITH_TLS "Enable TLS (Trough OpenSSL)" ON)

option(RESTC_CPP_LOG_WITH_INTERNAL_LOG "Use internal log handler" OFF)

option(RESTC_CPP_LOG_WITH_BOOST_LOG "Use boost::log for logging" OFF)

option(RESTC_CPP_LOG_WITH_LOGFAULT "Use logfault for logging" OFF)

option(RESTC_CPP_LOG_WITH_CLOG "Use std::clog for logging" OFF)

if (NOT RESTC_CPP_LOG_WITH_INTERNAL_LOG AND NOT RESTC_CPP_LOG_WITH_LOGFAULT AND NOT RESTC_CPP_LOG_WITH_BOOST_LOG AND NOT RESTC_CPP_LOG_WITH_CLOG)
    set(RESTC_CPP_LOG_WITH_INTERNAL_LOG ON)
endif()

set(RESTC_CPP_LOG_LEVEL_STR "debug" CACHE STRING "Limit logs to: none, error, warn, info, debug, trace")

option(RESTC_CPP_LOG_JSON_SERIALIZATION "Enable trace logging for json serialization debugging" OFF)

option(RESTC_CPP_WITH_ZLIB "Use zlib" ON)

option(RESTC_CPP_USE_CPP14 "Use the C++14 standard" OFF)

option(RESTC_CPP_USE_CPP17 "Use the C++17 standard" OFF)

option(RESTC_CPP_USE_CPP20 "Use the C++20 standard" OFF)

if (NOT RESTC_CPP_USE_CPP14 AND NOT RESTC_CPP_USE_CPP17 AND NOT RESTC_CPP_USE_CPP20)
    # Check if GCC 12 is being used and default it to C++17
    # https://www.mail-archive.com/debian-bugs-dist@lists.debian.org/msg1887728.html
    if (CMAKE_CXX_COMPILER_ID STREQUAL "GNU" AND CMAKE_CXX_COMPILER_VERSION VERSION_LESS 13.0 AND CMAKE_CXX_COMPILER_VERSION VERSION_GREATER_EQUAL 12.0)
        message(STATUS "GCC 12 detected, defaulting to C++17")
        set(RESTC_CPP_USE_CPP17 ON)
    elseif (COMPILER_SUPPORTS_CXX20)
        set(RESTC_CPP_USE_CPP20 ON)
    elseif (COMPILER_SUPPORTS_CXX17)
        set(RESTC_CPP_USE_CPP17 ON)
    else ()
        set(RESTC_CPP_USE_CPP14 ON)
    endif()
endif()

option(RESTC_CPP_THREADED_CTX "Allow asio contextx with multiple therads. Enables thread-safe internal access." OFF)

option(RESTC_CPP_SILENCE_BOOST_DEPRICATED_MESSAGES "Allows us to avoid warnings (mostly) because asio is not updated between boost versions" ON)
if(RESTC_CPP_SILENCE_BOOST_DEPRICATED_MESSAGES)
    add_definitions(-DBOOST_ALLOW_DEPRECATED_HEADERS=1)
endif()

if (NOT DEFINED RESTC_CPP_MAX_INPUT_BUFFER_LENGTH)
    set (RESTC_CPP_MAX_INPUT_BUFFER_LENGTH 0x7fffffff)
endif()

option(BOOST_ERROR_CODE_HEADER_ONLY "Work-around for another boost issue" ON)
if (BOOST_ERROR_CODE_HEADER_ONLY)
    add_definitions(
        -DBOOST_ERROR_CODE_HEADER_ONLY=1
        -DBOOST_SYSTEM_NO_DEPRECATED=1
        )
endif()

if (RESTC_CPP_LOG_LEVEL_STR STREQUAL "none")
    set (RESTC_CPP_LOG_LEVEL 0)
elseif(RESTC_CPP_LOG_LEVEL_STR STREQUAL  "error")
    set (RESTC_CPP_LOG_LEVEL 1)
elseif(RESTC_CPP_LOG_LEVEL_STR STREQUAL  "warn")
    set (RESTC_CPP_LOG_LEVEL 2)
elseif(RESTC_CPP_LOG_LEVEL_STR STREQUAL  "info")
    set (RESTC_CPP_LOG_LEVEL 3)
elseif(RESTC_CPP_LOG_LEVEL_STR STREQUAL  "debug")
    set (RESTC_CPP_LOG_LEVEL 4)
elseif(RESTC_CPP_LOG_LEVEL_STR STREQUAL  "trace")
    set (RESTC_CPP_LOG_LEVEL 5)
else()
    message(FATAL_ERROR "Unsupported log level ${RESTC_CPP_LOG_LEVEL_STR}")
endif()

if (RESTC_CPP_LOG_WITH_INTERNAL_LOG)
    if (RESTC_CPP_LOG_WITH_BOOST_LOG OR RESTC_CPP_LOG_WITH_LOGFAULT OR RESTC_CPP_LOG_WITH_CLOG)
        message( FATAL_ERROR "You must use RESTC_CPP_LOG_WITH_BOOST_LOG or RESTC_CPP_LOG_WITH_LOGFAULT or RESTC_CPP_LOG_WITH_INTERNAL_LOG or neither")
    endif()
    set(BOOST_LOG_DEP log)
    message(STATUS "Using internal log handler")
    set(LOGGING_SRC src/logging.cpp)
endif()

if (RESTC_CPP_LOG_WITH_BOOST_LOG)
    if (RESTC_CPP_LOG_WITH_LOGFAULT OR RESTC_CPP_LOG_WITH_CLOG OR RESTC_CPP_LOG_WITH_INTERNAL_LOG)
        message( FATAL_ERROR "You must use RESTC_CPP_LOG_WITH_BOOST_LOG or RESTC_CPP_LOG_WITH_LOGFAULT or RESTC_CPP_LOG_WITH_INTERNAL_LOG or neither")
    endif()
    set(BOOST_LOG_DEP log)
    message(STATUS "Using Boost.Log for logging (brace for horrors!)")
endif()

if (RESTC_CPP_LOG_WITH_LOGFAULT)
    if (RESTC_CPP_LOG_WITH_BOOST_LOG OR RESTC_CPP_LOG_WITH_CLOG OR RESTC_CPP_LOG_WITH_INTERNAL_LOG)
        message( FATAL_ERROR "You must use RESTC_CPP_LOG_WITH_BOOST_LOG or RESTC_CPP_LOG_WITH_LOGFAULT or RESTC_CPP_LOG_WITH_INTERNAL_LOG or neither")
    endif()
    message(STATUS "Using restc for logging")
endif()

if (RESTC_CPP_LOG_WITH_CLOG)
    if (RESTC_CPP_LOG_WITH_BOOST_LOG OR RESTC_CPP_LOG_WITH_LOGFAULT OR RESTC_CPP_LOG_WITH_INTERNAL_LOG)
        message( FATAL_ERROR "You must use RESTC_CPP_LOG_WITH_BOOST_LOG or RESTC_CPP_LOG_WITH_LOGFAULT or RESTC_CPP_LOG_WITH_INTERNAL_LOG or neither")
    endif()
    message(STATUS "Using std::clog for logging")
endif()

message(STATUS "Using ${CMAKE_CXX_COMPILER}")

macro(SET_CPP_STANDARD target)
    if (RESTC_CPP_USE_CPP20)
        message(STATUS "Using C++ 20 for ${target}")
        set_property(TARGET ${target} PROPERTY CXX_STANDARD 20)
    elseif (RESTC_CPP_USE_CPP17)
        message(STATUS "Using C++ 17 for ${target}")
        set_property(TARGET ${target} PROPERTY CXX_STANDARD 17)
    else()
        message(STATUS "Using C++ 14 for ${target}")
        set_property(TARGET ${target} PROPERTY CXX_STANDARD 14)
    endif()
endmacro(SET_CPP_STANDARD)

# We create a configuration file so that other code that include our header files gets the correct configuration.
CONFIGURE_FILE(config.h.template ${CMAKE_BINARY_DIR}/generated-include/${PROJECT_NAME}/config.h)

set(ACTUAL_SOURCES
    src/ChunkedReaderImpl.cpp
    src/ChunkedWriterImpl.cpp
    src/IoReaderImpl.cpp
    src/IoWriterImpl.cpp
    src/PlainReaderImpl.cpp
    src/PlainWriterImpl.cpp
    src/NoBodyReaderImpl.cpp
    src/DataReaderStream.cpp
    src/RestClientImpl.cpp
    src/RequestImpl.cpp
    src/ReplyImpl.cpp
    src/ConnectionPoolImpl.cpp
    src/Url.cpp
    src/RequestBodyStringImpl.cpp
    src/RequestBodyFileImpl.cpp
    src/url_encode.cpp
    src/boost_compitability.cpp
    ${LOGGING_SRC}
    )

set(HEADERS
    ${RESTC_ROOT_DIR}/include/restc-cpp/restc-cpp.h
    )

if (RESTC_CPP_WITH_ZLIB)
    set(ACTUAL_SOURCES ${ACTUAL_SOURCES} src/ZipReaderImpl.cpp)
endif()

set(SOURCES ${ACTUAL_SOURCES})

add_library(${PROJECT_NAME} ${SOURCES})
set_target_properties(${PROJECT_NAME} PROPERTIES DEBUG_OUTPUT_NAME restc-cppD)
target_include_directories(${PROJECT_NAME}
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_SOURCE_DIR}/include>
        $<BUILD_INTERFACE:${CMAKE_BINARY_DIR}/generated-include>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
    )

SET_CPP_STANDARD(${PROJECT_NAME})

if(NOT "${RESTC_CPP_EXTERNAL_DEPS}" STREQUAL "")
    add_dependencies(${PROJECT_NAME} ${RESTC_CPP_EXTERNAL_DEPS})
endif()

if (NOT EMBEDDED_RESTC_CPP)

    if (RESTC_USE_LEGACY_BOOST_FIND)
        unset(restc_cpp_boost_find_config)
        message("Using legacy Boost find config")
    elseif(CMAKE_VERSION VERSION_GREATER "3.28")
        set(restc_cpp_boost_find_config CONFIG)
        message("Using new Boost find config")
    endif()

    find_package(Boost ${RESTC_BOOST_VERSION} REQUIRED ${restc_cpp_boost_find_config} COMPONENTS
        system
        program_options
        filesystem
        date_time
        coroutine
        context
        chrono
        ${BOOST_LOG_DEP}
        )

    message(STATUS "Boost version found: ${Boost_VERSION}")

    target_include_directories(${PROJECT_NAME} PUBLIC ${Boost_INCLUDE_DIRS})
    target_link_libraries(${PROJECT_NAME} PUBLIC ${Boost_LIBRARIES})
    target_compile_definitions(${PROJECT_NAME} PUBLIC -DBOOST_COROUTINE_NO_DEPRECATION_WARNING=1)

    if (EXISTS ${Boost_INCLUDE_DIRS}/boost/type_index.hpp)
        set(RESTC_CPP_HAVE_BOOST_TYPEINDEX 1)
    endif()

    if (RESTC_CPP_WITH_TLS)
        find_package(OpenSSL REQUIRED)
        target_link_libraries(${PROJECT_NAME} PUBLIC ${OPENSSL_LIBRARIES})
        if (WIN32)
            target_link_libraries(${PROJECT_NAME} PUBLIC bcrypt)
        endif()
    endif()

    if (RESTC_CPP_WITH_ZLIB)
        find_package(ZLIB REQUIRED)
        target_include_directories(${PROJECT_NAME} PUBLIC ${ZLIB_INCLUDE_DIRS})
        target_link_libraries(${PROJECT_NAME} PUBLIC ${ZLIB_LIBRARIES})
    endif()

    if (UNIX)
        find_package(Threads REQUIRED)
        target_link_libraries(${PROJECT_NAME} PUBLIC ${CMAKE_THREAD_LIBS_INIT})
    endif()


    if (WIN32)
        link_directories(${Boost_LIBRARY_DIRS})
    endif()

    if(WIN32)
        add_definitions(-D_WIN32_WINNT=0x0600)
        set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} /SAFESEH:NO")
    endif()

    set(LIBRARY_OUTPUT_PATH ${CMAKE_BINARY_DIR}/lib CACHE PATH "Destination location")
    link_directories(${LIBRARY_OUTPUT_PATH})

    include(cmake_scripts/doxygen.cmake)

    target_include_directories(${PROJECT_NAME}
        PUBLIC
            $<BUILD_INTERFACE:${ZLIB_INCLUDE_DIR}>
            $<BUILD_INTERFACE:${ZLIB_INCLUDE_DIR}/build>
            $<BUILD_INTERFACE:${OPENSSL_INCLUDE_DIR}>
    )
endif()

include(cmake_scripts/install.cmake)
install(DIRECTORY ${CMAKE_BINARY_DIR}/generated-include/ DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})

if (RESTC_CPP_WITH_UNIT_TESTS OR RESTC_CPP_WITH_FUNCTIONALT_TESTS)
    enable_testing()
    add_subdirectory(tests)
endif()

if (RESTC_CPP_WITH_EXAMPLES)
    add_subdirectory(examples/logip)
endif()


