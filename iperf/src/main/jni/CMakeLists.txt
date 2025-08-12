# Modern minimum CMake that matches recent Android toolchains
cmake_minimum_required(VERSION 3.22)

project("iperf")

# Ensure PIC for all targets
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

#set(CMAKE_CXX_STANDARD 11)
set(IPERF_SRC ${CMAKE_SOURCE_DIR}/iperf3)

# check some architecture specific build options
if ("${ANDROID_ABI}" MATCHES "arm64-v8a|x86_64")
    set(HAVE_SO_MAX_PACING_RATE 1)
endif ()

# generate src/iperf_config.h
configure_file(${CMAKE_SOURCE_DIR}/../assets/iperf_config.h.in ${CMAKE_BINARY_DIR}/src/iperf_config.h)

# Make sure we can include the generated config header
include_directories(${CMAKE_BINARY_DIR}/src)

###################################

include_directories(${IPERF_SRC}/src)

# Creates and names a library, sets it as either STATIC
# or SHARED, and provides the relative paths to its source code.
# You can define multiple libraries, and CMake builds it for you.
# Gradle automatically packages shared libraries with your APK.

add_library( # build libiperf.a
        iperf
        STATIC

        ${IPERF_SRC}/src/cjson.c
        ${IPERF_SRC}/src/iperf_api.c
        ${IPERF_SRC}/src/iperf_error.c
        ${IPERF_SRC}/src/iperf_client_api.c
        ${IPERF_SRC}/src/iperf_locale.c
        ${IPERF_SRC}/src/iperf_server_api.c
        ${IPERF_SRC}/src/iperf_tcp.c
        ${IPERF_SRC}/src/iperf_udp.c
        ${IPERF_SRC}/src/iperf_sctp.c
        ${IPERF_SRC}/src/iperf_util.c
        ${IPERF_SRC}/src/net.c
        ${IPERF_SRC}/src/tcp_info.c
        ${IPERF_SRC}/src/timer.c
        ${IPERF_SRC}/src/units.c
        # Android pthread shims
        ${IPERF_SRC}/src/iperf_pthread.c
        # Missing sources for time and QoS parsing
        ${IPERF_SRC}/src/iperf_time.c
        ${IPERF_SRC}/src/dscp.c
        )

add_library(
        iperf-jni SHARED

        iperf-jni-wrap.cpp iperf-jni.cpp
)

# NDK/system libraries
find_library(
        log-lib
        log)

# Threads package maps to correct flags on Android
find_package(Threads REQUIRED)

# Link libraries
target_link_libraries(iperf PRIVATE Threads::Threads)

target_link_libraries(iperf-jni
        PRIVATE
        iperf
        Threads::Threads
        ${log-lib}
)

# Keep exported symbols minimal; JNI uses JNIEXPORT so this is safe
if (CMAKE_C_COMPILER_ID MATCHES "Clang|GNU")
    target_compile_options(iperf PRIVATE -fvisibility=hidden -fvisibility-inlines-hidden)
    target_compile_options(iperf-jni PRIVATE -fvisibility=hidden -fvisibility-inlines-hidden)
endif ()

# Compile definitions to ensure pthread functions are available
target_compile_definitions(iperf PRIVATE
        _GNU_SOURCE
        _POSIX_C_SOURCE=200809L
        HAVE_PTHREAD=1
)

target_compile_definitions(iperf-jni PRIVATE
        _GNU_SOURCE
        _POSIX_C_SOURCE=200809L
        HAVE_PTHREAD=1
)
