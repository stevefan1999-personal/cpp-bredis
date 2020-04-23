cmake_minimum_required(VERSION 3.0)
project (bredis CXX)

set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

include (CTest)

option(BREDIS_BUILD_EXAMPLES "Build examples?" OFF)

add_definitions(-DBOOST_ERROR_CODE_HEADER_ONLY)
add_definitions(-DBOOST_COROUTINES_NO_DEPRECATION_WARNING)
#add_definitions(-DBREDIS_DEBUG)

find_package(
    Boost
    COMPONENTS
        system
        coroutine
        filesystem
    REQUIRED
)

if(CMAKE_COMPILER_IS_GNUCXX)
    set(CMAKE_CXX_FLAGS "-g -O0 -fprofile-arcs -ftest-coverage")
    set(CMAKE_C_FLAGS "-g -O0 -fprofile-arcs -ftest-coverage")

    execute_process(COMMAND
        ${CMAKE_CXX_COMPILER} -dumpversion
        OUTPUT_VARIABLE GCC_VERSION)
    string(STRIP ${GCC_VERSION} GCC_VERSION)
    MESSAGE(STATUS "gcc version: [" ${GCC_VERSION} "]")

    # this works on gentoo, but not on travis/ubuntu, let's hard-code the value
    #execute_process(COMMAND
    #    "which" "gcov-${GCC_VERSION}"
    #    OUTPUT_VARIABLE GCOV_TOOL)
    #MESSAGE(STATUS "gcov: [" ${GCOV_TOOL} "]")
    #string(STRIP ${GCOV_TOOL} GCOV_TOOL)
    #MESSAGE(STATUS "gcov(fixed): [" ${GCOV_TOOL} "]")
    set(GCOV_TOOL "gcov-5")

    add_custom_target("coverage"
        COMMAND "lcov" --directory . --zerocounters
        COMMAND ctest
        COMMAND "lcov" --directory . --capture --output-file coverage.info --gcov-tool ${GCOV_TOOL}
        COMMAND "lcov" --remove coverage.info "'/t/*'" "'/boost_1_63_0/*'" "'/usr/*'" --output-file coverage.info.cleaned
        COMMAND "rm" coverage.info

        WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
    )
    add_custom_target("coverage-report"
        COMMAND "genhtml" -o coverage coverage.info.cleaned
        WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
        DEPENDS "coverage"
    )
endif()

add_library(
    bredis INTERFACE
)
target_include_directories(
    bredis
    INTERFACE ${bredis_SOURCE_DIR}/include ${Boost_INCLUDE_DIRS}
)
target_link_libraries(
    bredis
    INTERFACE ${Boost_LIBRARIES}
)
target_compile_definitions(
    bredis
    INTERFACE Boost_USE_MULTITHREADED
)

if (UNIX)
    #add_definitions(-Wall -Wextra -pedantic -Werror)
    add_definitions(-Wall -Wextra -pedantic)
    #add_definitions(-fsanitize=undefined -fsanitize=address -fno-omit-frame-pointer)
    target_link_libraries(bredis INTERFACE pthread) #  asan ubsan
endif()

if ((Boost_MAJOR_VERSION GREATER_EQUAL 1) AND (Boost_MINOR_VERSION GREATER_EQUAL 70))
    set(BOOST_VERSION_OK true)
else()
    set(BOOST_VERSION_OK false)
endif()

if (BREDIS_BUILD_EXAMPLES)
    add_subdirectory(examples)
endif()

if (BUILD_TESTING)
    enable_testing()
    add_library(catch_lib t/tests-main.cpp)

    if (WIN32)
        target_compile_definitions(bredis INTERFACE _WIN32_WINNT=0x0601)
    endif()

    add_executable(t-05-protocol t/05-protocol.cpp)
    target_link_libraries(t-05-protocol bredis catch_lib)
    add_test("t-05-protocol" t-05-protocol)

    add_executable(t-06-fragmented-buffer t/06-fragmented-buffer.cpp)
    target_link_libraries(t-06-fragmented-buffer bredis catch_lib)
    add_test("t-06-fragmented-buffer" t-06-fragmented-buffer)

    add_executable(t-07-extract t/07-extract.cpp)
    target_link_libraries(t-07-extract bredis catch_lib)
    add_test("t-07-extract" t-07-extract)

    add_executable(t-08-markers t/08-markers.cpp)
    target_link_libraries(t-08-markers bredis catch_lib)
    add_test("t-07-extract" t-08-markers)

    add_executable(t-10-ping t/10-ping.cpp)
    target_link_libraries(t-10-ping bredis catch_lib)
    add_test("t-10-ping" t-10-ping)

    add_executable(t-11-multi-ping t/11-multi-ping.cpp)
    target_link_libraries(t-11-multi-ping bredis catch_lib)
    add_test("t-11-multi-ping" t-11-multi-ping)

    add_executable(t-12-basic-types t/12-basic-types.cpp)
    target_link_libraries(t-12-basic-types bredis catch_lib)
    add_test("t-12-basic-types" t-12-basic-types)

    add_executable(t-13-protol-error t/13-protol-error.cpp)
    target_link_libraries(t-13-protol-error bredis catch_lib)
    add_test("t-13-protol-error" t-13-protol-error)

    if(NOT WIN32)
        add_executable(t-14-uds t/14-uds.cpp)
        target_link_libraries(t-14-uds bredis catch_lib)
        add_test("t-14-uds" t-14-uds)
    endif()

    add_executable(t-15-cancellation t/15-cancellation.cpp)
    target_link_libraries(t-15-cancellation bredis catch_lib)
    add_test("t-15-cancellation" t-15-cancellation)

    add_executable(t-16-close-connection t/16-close-connection.cpp)
    target_link_libraries(t-16-close-connection bredis catch_lib)
    add_test("t-16-close-connection" t-16-close-connection)

    add_executable(t-17-sync t/17-sync.cpp)
    target_link_libraries(t-17-sync bredis catch_lib)
    add_test("t-17-sync" t-17-sync)

    add_executable(t-18-subscription t/18-subscription.cpp)
    target_link_libraries(t-18-subscription bredis catch_lib)
    add_test("t-18-subscription" t-18-subscription)

    add_executable(t-19-transaction t/19-transaction.cpp)
    target_link_libraries(t-19-transaction bredis catch_lib)
    add_test("t-19-transaction" t-19-transaction)

    if(BOOST_VERSION_OK)
        add_executable(t-20-promise t/20-promise.cpp)
        target_link_libraries(t-20-promise bredis catch_lib)
        add_test("t-20-promise" t-20-promise)
    endif()

    add_executable(t-21-coroutine t/21-coroutine.cpp)
    target_link_libraries(t-21-coroutine bredis catch_lib)
    add_test("t-21-coroutine" t-21-coroutine)

    add_executable(t-22-ping_drop-policy t/22-ping_drop-policy.cpp)
    target_link_libraries(t-22-ping_drop-policy bredis catch_lib)
    add_test("t-22-ping_drop-policy" t-22-ping_drop-policy)

    add_executable(t-23-stream t/23-stream.cpp)
    target_link_libraries(t-23-stream bredis catch_lib)
    add_test("t-23-stream" t-23-stream)

    if(BOOST_VERSION_OK)
        add_executable(t-24-dynbuff t/24-dynbuff.cpp)
        target_link_libraries(t-24-dynbuff bredis catch_lib)
        add_test("t-24-dynbuff" t-24-dynbuff)
    endif()
endif()