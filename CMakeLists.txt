project(arnold-usd)

cmake_minimum_required(VERSION 3.12)

add_compile_options($<$<CXX_COMPILER_ID:MSVC>:/MP>)

if (UNIX AND NOT APPLE)
    set(LINUX TRUE)
endif ()

if (LINUX)
    set(LIB_EXTENSION ".so")
    add_compile_definitions(_LINUX)
elseif (APPLE)
    set(LIB_EXTENSION ".dylib")
    add_compile_definitions(_DARWIN)
else ()
    set(LIB_EXTENSION ".dll")
    add_compile_definitions(_WINDOWS _WIN32 WIN32)
    add_compile_definitions(_WIN64)
endif ()

list(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake/modules" 0)
list(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake/utils" 0)

include(options)

# Global required packagse
set(LIBDIR  ${CMAKE_SOURCE_DIR}/../lib/win64_vc15)

set(USD_INCLUDE_DIR  ${LIBDIR}/usd-debug/include)
set(USD_LIBRARY_DIR  ${LIBDIR}/usd-debug/lib)

set(Boost_INCLUDE_DIR ${LIBDIR}/usd-debug/include/boost-1_70)
set(Boost_LIBRARY_DIR ${LIBDIR}/usd-debug/lib)

if(CMAKE_BUILD_TYPE MATCHES "Release")

else()
#   set(USD_INCLUDE_DIR  ${LIBDIR}/usd-debug-debug/include)
#   set(USD_LIBRARY_DIR  ${LIBDIR}/usd-debug-debug/lib)
endif()

find_package(Arnold REQUIRED)

if (NOT BUILD_USE_CUSTOM_BOOST)
    if (USD_HAS_PYTHON)
        # find_package(Boost COMPONENTS python REQUIRED)
    else ()
        # find_package(Boost REQUIRED)
    endif ()
endif ()
if (BUILD_SCHEMAS)
    find_package(Python2 COMPONENTS Development Interpreter REQUIRED)
else ()
    set(PYTHON_INCLUDE_DIRS ${LIBDIR}/python/37/include)
    set(PYTHON_LIB ${LIBDIR}/python/37/libs/python37.lib)
    set(PYTHON_EXECUTABLE ${LIBDIR}/python/37/python.exe)
endif ()

set(TBB_INCLUDE_DIRS ${LIBDIR}/usd-debug/include)
list(APPEND TBB_LIBRARIES
  ${LIBDIR}/usd-debug/lib/tbb.lib
  ${LIBDIR}/usd-debug/lib/tbb_debug.lib
  ${LIBDIR}/usd-debug/lib/tbb_preview.lib
  ${LIBDIR}/usd-debug/lib/tbb_preview_debug.lib
  ${LIBDIR}/usd-debug/lib/tbbmalloc.lib
  ${LIBDIR}/usd-debug/lib/tbbmalloc_debug.lib
  ${LIBDIR}/usd-debug/lib/tbbmalloc_proxy.lib
  ${LIBDIR}/usd-debug/lib/tbbmalloc_proxy_debug.lib
  ${LIBDIR}/usd-debug/lib/tbbproxy.lib
  ${LIBDIR}/usd-debug/lib/tbbproxy_debug.lib
)
add_definitions(-DTBB_USE_THREADING_TOOLS)
add_definitions(-DTBB_USE_ASSERT)

include_directories(
    ${USD_INCLUDE_DIR}
    ${Boost_INCLUDE_DIR}
    ${PYTHON_INCLUDE_DIRS}
    ${TBB_INCLUDE_DIRS}
)

link_directories(
    ${USD_LIBRARY_DIR}
    ${Boost_LIBRARY_DIR}
    ${LIBDIR}/python/37/libs
)

set(USD_VERSION  "0.20.11")
set(USD_MAJOR_VERSION 20)
set(USD_MINOR_VERSION 11)
set(USD_PATCH_VERSION 1)
set(USD_HAS_FULLSCREEN_SHADER ON)


configure_file(
    "${CMAKE_CURRENT_SOURCE_DIR}/arnold_usd.h.in"
    "${CMAKE_CURRENT_BINARY_DIR}/arnold_usd.h")

set(CMAKE_CXX_STANDARD 11 CACHE STRING "CMake CXX Standard")

if (APPLE)
    string(APPEND CMAKE_SHARED_LINKER_FLAGS " -Wl,-undefined,error")
elseif (LINUX)
    string(APPEND CMAKE_SHARED_LINKER_FLAGS " -Wl,--no-undefined")
endif ()

if (WIN32)
    # So we can use std::min and std::max, because windows headers are indirectly included by TF.
    add_compile_definitions(NOMINMAX)
    if (TBB_NO_EXPLICIT_LINKAGE)
        add_compile_definitions(__TBB_NO_IMPLICIT_LINKAGE=1)
    endif ()
endif ()

if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
    if (CMAKE_CXX_COMPILER_VERSION VERSION_GREATER_EQUAL 6.3)
        # GCC 6.3.1 complains about the usage of auto_ptr from the newer
        # TBB versions.
        add_compile_options(-Wno-deprecated-declarations)
        if (BUILD_DISABLE_CXX11_ABI)
            add_compile_options(-D_GLIBCXX_USE_CXX11_ABI=0)
        endif ()
    endif ()
endif ()

if (NOT USD_STATIC_BUILD)
    if (BUILD_SCHEMAS AND USD_HAS_PYTHON)
        add_subdirectory(schemas)
        install(FILES plugInfo.json
                DESTINATION "${PREFIX_LIB}/usd")
    endif ()

    if (BUILD_NDR_PLUGIN)
        add_subdirectory(ndr)
    endif ()

    if (BUILD_RENDER_DELEGATE)
        add_subdirectory(render_delegate)
    endif ()

    if (BUILD_USD_IMAGING_PLUGIN)
        add_subdirectory(usd_imaging)
    endif ()

    if (BUILD_RENDER_DELEGATE OR BUILD_NDR_PLUGIN OR BUILD_USD_IMAGING_PLUGIN)
        install(FILES plugInfo.json
            DESTINATION "${PREFIX_PLUGINS}")
    endif ()
endif ()

if (BUILD_PROCEDURAL OR BUILD_USD_WRITER)
    add_subdirectory(translator)
endif ()

if (BUILD_PROCEDURAL)
    add_subdirectory(procedural)
endif ()

if (BUILD_USD_WRITER)
    add_subdirectory(cmd)
endif ()

if (BUILD_DOCS)
    add_subdirectory(docs)
endif ()

if (BUILD_TESTSUITE)
    enable_testing()
    add_subdirectory(testsuite)
endif ()

install(FILES LICENSE.md
    DESTINATION .)

install(FILES "${CMAKE_CURRENT_BINARY_DIR}/arnold_usd.h"
    DESTINATION "${PREFIX_HEADERS}")
