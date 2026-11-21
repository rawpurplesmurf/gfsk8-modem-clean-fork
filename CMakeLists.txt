cmake_minimum_required(VERSION 3.20)
project(gfsk8modem C CXX)
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_C_STANDARD 11)

option(GFSK8_BUILD_APPS "Build optional PortAudio demo applications (js8rx, js8tx)" OFF)

# ── KissFFT ────────────────────────────────────────────────────────────────────
add_library(kissfft STATIC
    vendor/kissfft/kiss_fft.c
    vendor/kissfft/kiss_fftr.c
)
target_include_directories(kissfft PUBLIC vendor)
target_compile_options(kissfft PRIVATE -Wall -Wextra)
set(KISSFFT_LIB kissfft)
set(KISSFFT_INCLUDE "${CMAKE_CURRENT_SOURCE_DIR}/vendor")

# ── Library ───────────────────────────────────────────────────────────────────
add_library(gfsk8modem STATIC
    src/js8codec.cpp
    src/JS8.cpp
    src/JS8Submode.cpp
    src/Varicode.cpp
    src/JSC.cpp
    src/JSC_map.cpp
    src/JSC_list.cpp
    src/DecodedText.cpp
    src/api.cpp
)

target_include_directories(gfsk8modem
    PUBLIC  include
    PRIVATE src
    PRIVATE vendor
)

target_link_libraries(gfsk8modem
    PRIVATE ${KISSFFT_LIB}
)

target_compile_options(gfsk8modem PRIVATE
    -Wall -Wextra
)

# ── Tests ─────────────────────────────────────────────────────────────────────
enable_testing()

if(EXISTS "${CMAKE_SOURCE_DIR}/cpp_tests/test_encode.cpp")
    add_executable(test_encode cpp_tests/test_encode.cpp)
    target_include_directories(test_encode PRIVATE vendor)
    target_link_libraries(test_encode PRIVATE gfsk8modem ${KISSFFT_LIB})
    add_test(NAME encode COMMAND test_encode)
endif()

if(EXISTS "${CMAKE_SOURCE_DIR}/cpp_tests/test_loopback.cpp")
    add_executable(test_loopback cpp_tests/test_loopback.cpp)
    target_link_libraries(test_loopback PRIVATE gfsk8modem)
    add_test(NAME loopback COMMAND test_loopback)
endif()

if(EXISTS "${CMAKE_SOURCE_DIR}/cpp_tests/test_loopback_all.cpp")
    add_executable(test_loopback_all cpp_tests/test_loopback_all.cpp)
    target_link_libraries(test_loopback_all PRIVATE gfsk8modem)
    add_test(NAME loopback_all COMMAND test_loopback_all)
endif()

if(EXISTS "${CMAKE_SOURCE_DIR}/cpp_tests/test_decode_wav.cpp")
    add_executable(test_decode_wav cpp_tests/test_decode_wav.cpp)
    target_link_libraries(test_decode_wav PRIVATE gfsk8modem)
endif()

# ── Optional Demo Apps (Disabled by Default) ──────────────────────────────────
if(GFSK8_BUILD_APPS)
    find_package(PkgConfig QUIET)
    if(PkgConfig_FOUND)
        pkg_check_modules(PORTAUDIO QUIET portaudio-2.0)
        if(NOT PORTAUDIO_FOUND)
            pkg_check_modules(PORTAUDIO QUIET portaudio)
        endif()
    endif()

    if(PORTAUDIO_FOUND AND EXISTS "${CMAKE_SOURCE_DIR}/apps/js8rx/main.cpp")
        add_executable(js8rx apps/js8rx/main.cpp)
        target_include_directories(js8rx PRIVATE ${PORTAUDIO_INCLUDE_DIRS})
        target_link_libraries(js8rx PRIVATE gfsk8modem ${PORTAUDIO_LIBRARIES})
        target_compile_options(js8rx PRIVATE -Wall -Wextra)
        message(STATUS "js8rx: PortAudio found via pkg-config, building js8rx")
    else()
        message(STATUS "js8rx: PortAudio not found, skipping js8rx")
    endif()

    if(PORTAUDIO_FOUND AND EXISTS "${CMAKE_SOURCE_DIR}/apps/js8tx/main.cpp")
        add_executable(js8tx apps/js8tx/main.cpp)
        target_include_directories(js8tx PRIVATE ${PORTAUDIO_INCLUDE_DIRS})
        target_link_libraries(js8tx PRIVATE gfsk8modem ${PORTAUDIO_LIBRARIES})
        target_compile_options(js8tx PRIVATE -Wall -Wextra)
        message(STATUS "js8tx: PortAudio found via pkg-config, building js8tx")
    else()
        message(STATUS "js8tx: PortAudio not found, skipping js8tx")
    endif()
endif()
