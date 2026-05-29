cmake_minimum_required(VERSION 3.10)

# Parse version from gif_lib.h so it stays the single source of truth.
file(READ "${CMAKE_CURRENT_SOURCE_DIR}/gif_lib.h" _gif_lib_h)
string(REGEX MATCH "#define GIFLIB_MAJOR[ \t]+([0-9]+)" _ "${_gif_lib_h}")
set(GIFLIB_MAJOR ${CMAKE_MATCH_1})
string(REGEX MATCH "#define GIFLIB_MINOR[ \t]+([0-9]+)" _ "${_gif_lib_h}")
set(GIFLIB_MINOR ${CMAKE_MATCH_1})
string(REGEX MATCH "#define GIFLIB_RELEASE[ \t]+([0-9]+)" _ "${_gif_lib_h}")
set(GIFLIB_RELEASE ${CMAKE_MATCH_1})

project(giflib
    VERSION ${GIFLIB_MAJOR}.${GIFLIB_MINOR}.${GIFLIB_RELEASE}
    LANGUAGES C
)

set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED ON)

# Force static everywhere: project libs are static archives, and on platforms
# that support it the executables are linked fully statically too.
set(BUILD_SHARED_LIBS OFF)

if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "" FORCE)
endif()

if(MSVC)
    add_compile_options(/W3)
    # Use the static CRT so the resulting binaries have no MSVCRT runtime dep.
    set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
else()
    add_compile_options(-Wall)
endif()

# -------------------------------------------------------------------
# libgif (static)
# -------------------------------------------------------------------
set(GIF_SOURCES
    dgif_lib.c
    egif_lib.c
    gifalloc.c
    gif_err.c
    gif_font.c
    gif_hash.c
    openbsd-reallocarray.c
    quantize.c
)

add_library(gif STATIC ${GIF_SOURCES})
set_target_properties(gif PROPERTIES
    OUTPUT_NAME gif
    POSITION_INDEPENDENT_CODE ON
)
target_include_directories(gif PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})

find_library(M_LIBRARY m)
if(M_LIBRARY)
    target_link_libraries(gif PUBLIC ${M_LIBRARY})
endif()

# -------------------------------------------------------------------
# libutil (static) — shared CLI helpers used by every utility
# -------------------------------------------------------------------
add_library(gifutil STATIC
    qprintf.c
    getarg.c
)
set_target_properties(gifutil PROPERTIES
    OUTPUT_NAME util
    POSITION_INDEPENDENT_CODE ON
)
target_include_directories(gifutil PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})
target_link_libraries(gifutil PUBLIC gif)

# -------------------------------------------------------------------
# Fully-static executables
# -------------------------------------------------------------------
# On Linux/MinGW we can ask the linker for a fully static binary
# (libc, libm, etc. baked in). Apple does not ship a static libSystem,
# so on macOS the project libs are static but system libs stay dynamic.
if(CMAKE_SYSTEM_NAME STREQUAL "Linux"
        OR CMAKE_SYSTEM_NAME STREQUAL "FreeBSD"
        OR MINGW)
    set(GIFLIB_FULLY_STATIC_LINK ON)
else()
    set(GIFLIB_FULLY_STATIC_LINK OFF)
endif()

function(add_gif_tool name)
    add_executable(${name} ${name}.c)
    target_link_libraries(${name} PRIVATE gifutil gif)
    if(GIFLIB_FULLY_STATIC_LINK AND NOT MSVC)
        target_link_options(${name} PRIVATE -static)
    endif()
endfunction()

set(GIFLIB_TOOLS
    gifbuild
    giffix
    giftext
    giftool
    gifclrmp
    gifbg
    gifcolor
    gifecho
    giffilter
    gifhisto
    gifinto
    gifsponge
    gifwedge
    gif2rgb
)

foreach(tool ${GIFLIB_TOOLS})
    add_gif_tool(${tool})
endforeach()

# Tools that the upstream Makefile actually installs.
set(GIFLIB_INSTALLABLE_TOOLS
    gifbuild
    giffix
    giftext
    giftool
    gifclrmp
)

# -------------------------------------------------------------------
# Install
# -------------------------------------------------------------------
include(GNUInstallDirs)

install(TARGETS gif gifutil
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
)
install(FILES gif_lib.h DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
install(TARGETS ${GIFLIB_INSTALLABLE_TOOLS}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)
