# =============================================================================
#                          BLAKE3 交叉编译 CMake 构建系统
# =============================================================================
#
# 作者: 连思鑫（基于官方 BLAKE3 CMakeLists.txt 进行大幅优化与增强）
#
# 目标与功能：
#   • 支持 BLAKE3 的 C 实现编译为静态库或动态库
#   • 自动检测并启用 SIMD 加速（SSE2/SSE4.1/AVX2/AVX-512/NEON）
#   • 支持多平台交叉编译：x86 / x64 / arm32 / arm64
#   • 支持 Linux（原生 & 交叉）与 Windows（MinGW 交叉 & MSVC 原生）
#   • 可选启用 Intel oneTBB 并行版本（blake3_tbb.cpp）
#   • Windows 平台下静态库强制输出 .lib（而非 .a）
#
# 使用方式示例：
#
#   # Linux x64 静态库（原生）
#   cmake -B build -S . -DCOMPILER_TARGET=x64 -DTARGET_OS=linux -DTARGET_TYPE=static
#   cmake --build build -j$(nproc)
#
#   # Windows x64 静态库（MinGW 交叉编译，从 Linux 主机）
#   cmake -B build-win-x64 -S . -DCOMPILER_TARGET=x64 -DTARGET_OS=windows -DTARGET_TYPE=static
#
#   # Linux arm64 静态库（交叉编译）
#   cmake -B build-arm64 -S . -DCOMPILER_TARGET=arm64 -DTARGET_OS=linux -DTARGET_TYPE=static
#
#   # Linux arm32 静态库（带 NEON）
#   cmake -B build-arm32 -S . -DCOMPILER_TARGET=arm32 -DTARGET_OS=linux -DTARGET_TYPE=static
#
# =============================================================================

cmake_minimum_required(VERSION 4.1)

project(libblake3
        VERSION 1.5.1   # 请根据实际使用的 BLAKE3 版本修改
        DESCRIPTION "BLAKE3 hash function library with multi-platform & SIMD support"
        LANGUAGES C ASM
)

# ─────────────────────────────────────────────────────────────────────────────
#  1. 全局构建类型默认值
# ─────────────────────────────────────────────────────────────────────────────
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
endif()

set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS OFF)

# ─────────────────────────────────────────────────────────────────────────────
#  2. 用户可配置选项
# ─────────────────────────────────────────────────────────────────────────────

# 目标操作系统（影响文件名、定义、链接行为）
set(TARGET_OS "linux" CACHE STRING "Target operating system: linux | windows")
set_property(CACHE TARGET_OS PROPERTY STRINGS linux windows)

# 目标架构（决定 SIMD 策略与交叉编译器）
set(COMPILER_TARGET "x64" CACHE STRING "Target CPU architecture")
set_property(CACHE COMPILER_TARGET PROPERTY STRINGS x86 x64 arm32 arm64)

# 库类型：static / shared
set(TARGET_TYPE "static" CACHE STRING "Library type")
set_property(CACHE TARGET_TYPE PROPERTY STRINGS static shared)

option(ENABLE_TBB          "Enable oneTBB (Threading Building Blocks) for parallel hashing" OFF)
option(ENABLE_ASSEMBLY     "Enable hand-written assembly optimizations (x64 only)"         ON)

# 根据用户选择设置 BUILD_SHARED_LIBS（CMake 内部变量）
set(BUILD_SHARED_LIBS OFF)
if(TARGET_TYPE STREQUAL "shared")
    set(BUILD_SHARED_LIBS ON)
endif()

# ─────────────────────────────────────────────────────────────────────────────
#  3. 架构 → CMake 系统处理器映射 + BLAKE3 内部识别字符串
# ─────────────────────────────────────────────────────────────────────────────

if(COMPILER_TARGET STREQUAL "x64")
    set(CMAKE_SYSTEM_PROCESSOR x86_64)
    set(BLAKE3_TARGET_ARCH     amd64)
elseif(COMPILER_TARGET STREQUAL "x86")
    set(CMAKE_SYSTEM_PROCESSOR i686)
    set(BLAKE3_TARGET_ARCH     x86)
elseif(COMPILER_TARGET STREQUAL "arm64")
    set(CMAKE_SYSTEM_PROCESSOR aarch64)
    set(BLAKE3_TARGET_ARCH     arm64)
elseif(COMPILER_TARGET STREQUAL "arm32")
    set(CMAKE_SYSTEM_PROCESSOR arm)
    set(BLAKE3_TARGET_ARCH     arm)
else()
    message(FATAL_ERROR "Unsupported COMPILER_TARGET: ${COMPILER_TARGET}")
endif()

# ─────────────────────────────────────────────────────────────────────────────
#  4. 交叉编译器与平台特定编译选项
# ─────────────────────────────────────────────────────────────────────────────

if(TARGET_OS STREQUAL "windows")
    # ───── Windows 目标 ─────
    set(CMAKE_SYSTEM_NAME Windows)

    if(COMPILER_TARGET STREQUAL "x64")
        set(CMAKE_C_COMPILER   x86_64-w64-mingw32-gcc)
        set(CMAKE_ASM_COMPILER x86_64-w64-mingw32-gcc)
    elseif(COMPILER_TARGET STREQUAL "x86")
        set(CMAKE_C_COMPILER   i686-w64-mingw32-gcc)
        set(CMAKE_ASM_COMPILER i686-w64-mingw32-gcc)
    elseif(COMPILER_TARGET STREQUAL "arm64")
        set(CMAKE_C_COMPILER   aarch64-w64-mingw32-gcc)
        set(CMAKE_ASM_COMPILER aarch64-w64-mingw32-gcc)
    endif()

    add_definitions(-DWIN32 -D_WIN32)

elseif(TARGET_OS STREQUAL "linux")
    # ───── Linux 目标 ─────
    if(COMPILER_TARGET STREQUAL "arm64")
        set(CMAKE_C_COMPILER   aarch64-linux-gnu-gcc)
        set(CMAKE_ASM_COMPILER aarch64-linux-gnu-gcc)
        # arm64 默认支持 NEON，无需额外 -mfpu=neon

    elseif(COMPILER_TARGET STREQUAL "arm32")
        set(CMAKE_C_COMPILER   arm-linux-gnueabihf-gcc)
        set(CMAKE_ASM_COMPILER arm-linux-gnueabihf-gcc)
        # armv7 + NEON + hard float（大多数现代 arm32 Linux 发行版使用 hard float）
        add_compile_options(-march=armv7-a -mfpu=neon-vfpv4 -mfloat-abi=hard)

    elseif(COMPILER_TARGET STREQUAL "x86")
        add_compile_options(-m32)
        set(CMAKE_EXE_LINKER_FLAGS    "${CMAKE_EXE_LINKER_FLAGS} -m32")
        set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -m32")
    endif()

else()
    message(FATAL_ERROR "Unsupported TARGET_OS: ${TARGET_OS}")
endif()

# ─────────────────────────────────────────────────────────────────────────────
#  5. SIMD 策略选择
# ─────────────────────────────────────────────────────────────────────────────

set(BLAKE3_SIMD_TYPE "portable")

if(ENABLE_ASSEMBLY AND COMPILER_TARGET STREQUAL "x64")
    set(BLAKE3_SIMD_TYPE "amd64-asm")
elseif(COMPILER_TARGET MATCHES "^(x86|x64)$")
    set(BLAKE3_SIMD_TYPE "x86-intrinsics")
elseif(COMPILER_TARGET MATCHES "^(arm32|arm64)$")
    set(BLAKE3_SIMD_TYPE "neon-intrinsics")
endif()

# ─────────────────────────────────────────────────────────────────────────────
#  6. 源文件收集
# ─────────────────────────────────────────────────────────────────────────────

set(SOURCES
        blake3/blake3.c
        blake3/blake3_dispatch.c
        blake3/blake3_portable.c
)

# 根据 SIMD 类型添加对应实现
if(BLAKE3_SIMD_TYPE STREQUAL "amd64-asm")
    if(TARGET_OS STREQUAL "windows")
        list(APPEND SOURCES
                blake3/blake3_sse2_x86-64_windows_gnu.S
                blake3/blake3_sse41_x86-64_windows_gnu.S
                blake3/blake3_avx2_x86-64_windows_gnu.S
                blake3/blake3_avx512_x86-64_windows_gnu.S
        )
    else()
        list(APPEND SOURCES
                blake3/blake3_sse2_x86-64_unix.S
                blake3/blake3_sse41_x86-64_unix.S
                blake3/blake3_avx2_x86-64_unix.S
                blake3/blake3_avx512_x86-64_unix.S
        )
    endif()

elseif(BLAKE3_SIMD_TYPE STREQUAL "x86-intrinsics")
    list(APPEND SOURCES
            blake3/blake3_sse2.c
            blake3/blake3_sse41.c
            blake3/blake3_avx2.c
            blake3/blake3_avx512.c
    )

elseif(BLAKE3_SIMD_TYPE STREQUAL "neon-intrinsics")
    list(APPEND SOURCES blake3/blake3_neon.c)
endif()

# TBB 并行版本（C++ 文件）
if(ENABLE_TBB)
    find_package(TBB QUIET)
    if(TBB_FOUND)
        list(APPEND SOURCES blake3/blake3_tbb.cpp)
        set(USE_TBB ON)
    else()
        message(WARNING "TBB not found → parallel support disabled")
        set(USE_TBB OFF)
    endif()
else()
    set(USE_TBB OFF)
endif()

# ─────────────────────────────────────────────────────────────────────────────
#  7. 创建库目标
# ─────────────────────────────────────────────────────────────────────────────

add_library(blake3 ${SOURCES})

# Windows 平台文件名风格调整（尤其是静态库强制 .lib）
if(TARGET_OS STREQUAL "windows")
    if(BUILD_SHARED_LIBS)
        target_compile_definitions(blake3 PRIVATE BLAKE3_DLL_EXPORTS)
        target_compile_definitions(blake3 PUBLIC  BLAKE3_DLL)
        set_target_properties(blake3 PROPERTIES
                PREFIX ""
                SUFFIX ".dll"
                ARCHIVE_OUTPUT_NAME "blake3"   # import lib → blake3.lib
                RUNTIME_OUTPUT_NAME "blake3"   # dll → blake3.dll
        )
    else()
        # 静态库强制输出 blake3.lib （MinGW 默认会出 libblake3.a）
        set_target_properties(blake3 PROPERTIES
                PREFIX ""
                SUFFIX ".lib"
                OUTPUT_NAME "blake3"
        )
    endif()
else()
    # Linux / Unix 风格
    set_target_properties(blake3 PROPERTIES
            OUTPUT_NAME "blake3"
            PREFIX "lib"
    )
    if(BUILD_SHARED_LIBS)
        set_target_properties(blake3 PROPERTIES SUFFIX ".so")
    else()
        set_target_properties(blake3 PROPERTIES SUFFIX ".a")
    endif()
endif()

# 包含路径 & 符号可见性
target_include_directories(blake3 PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/blake3>
        $<INSTALL_INTERFACE:include>
)

set_target_properties(blake3 PROPERTIES C_VISIBILITY_PRESET hidden)

# ─────────────────────────────────────────────────────────────────────────────
#  8. SIMD 专用编译选项（逐文件设置，避免全局污染）
# ─────────────────────────────────────────────────────────────────────────────

if(BLAKE3_SIMD_TYPE STREQUAL "x86-intrinsics")
    if(CMAKE_C_COMPILER_ID MATCHES "GNU|Clang")
        set_source_files_properties(blake3/blake3_sse2.c   PROPERTIES COMPILE_FLAGS "-msse2")
        set_source_files_properties(blake3/blake3_sse41.c  PROPERTIES COMPILE_FLAGS "-msse4.1")
        set_source_files_properties(blake3/blake3_avx2.c   PROPERTIES COMPILE_FLAGS "-mavx2")
        set_source_files_properties(blake3/blake3_avx512.c PROPERTIES COMPILE_FLAGS "-mavx512f -mavx512vl")
    endif()

elseif(BLAKE3_SIMD_TYPE STREQUAL "neon-intrinsics")
    # 只对 arm32 增加 NEON 选项，arm64 默认支持无需指定
    if(COMPILER_TARGET STREQUAL "arm32" AND CMAKE_C_COMPILER_ID MATCHES "GNU|Clang")
        set_source_files_properties(blake3/blake3_neon.c PROPERTIES
                COMPILE_FLAGS "-mfpu=neon-vfpv4 -mfloat-abi=hard"
        )
    endif()
endif()

# TBB 链接与定义
if(USE_TBB)
    target_link_libraries(blake3 PUBLIC TBB::tbb)
    target_compile_definitions(blake3 PUBLIC BLAKE3_USE_TBB)
endif()

# ─────────────────────────────────────────────────────────────────────────────
#  9. 安装规则
# ─────────────────────────────────────────────────────────────────────────────

install(TARGETS blake3
        ARCHIVE DESTINATION lib
        LIBRARY DESTINATION lib
        RUNTIME DESTINATION bin
)

install(FILES blake3/blake3.h DESTINATION include)

# ─────────────────────────────────────────────────────────────────────────────
# 10. 构建完成信息汇总（便于调试）
# ─────────────────────────────────────────────────────────────────────────────

set(lib_type_desc "static")
if(BUILD_SHARED_LIBS)
    set(lib_type_desc "shared")
endif()

message(STATUS "")
message(STATUS "┌───────────────────────────────────────────────────────────────┐")
message(STATUS "│                  BLAKE3 构建配置汇总                          │")
message(STATUS "├───────────────────────────────────────────────────────────────┤")
message(STATUS "│  Target OS       : ${TARGET_OS}")
message(STATUS "│  Target Arch     : ${COMPILER_TARGET} (${CMAKE_SYSTEM_PROCESSOR})")
message(STATUS "│  Library Type    : ${TARGET_TYPE} (${lib_type_desc})")
message(STATUS "│  SIMD Strategy   : ${BLAKE3_SIMD_TYPE}")
message(STATUS "│  TBB Parallel    : ${USE_TBB}")
message(STATUS "│  Build Type      : ${CMAKE_BUILD_TYPE}")
message(STATUS "│  C Compiler      : ${CMAKE_C_COMPILER}")
message(STATUS "└───────────────────────────────────────────────────────────────┘")
message(STATUS "")