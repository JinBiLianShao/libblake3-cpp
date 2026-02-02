# 📘 BLAKE3 交叉编译 CMake 构建系统
# 作者: 连思鑫 (基于BLAKE3官方CMake配置修改)
# 
# 本构建系统专为编译BLAKE3哈希算法库设计，支持以下功能：
#   ✅ 编译BLAKE3 C实现为静态库或动态库
#   ✅ 自动检测并启用SIMD加速 (SSE4.1/AVX2/AVX512/NEON)
#   ✅ 支持多平台交叉编译 (x86/x64/arm32/arm64)
#   ✅ 支持Windows (MinGW/MSVC) 和 Linux 平台
#   ✅ 可选TBB并行计算支持
#
# 📁 目录结构要求：
#   blake3_project/
#   ├── CMakeLists.txt        (此文件)
#   ├── blake3/               # BLAKE3 C源码目录
#   │   ├── blake3.c
#   │   ├── blake3.h
#   │   ├── blake3_impl.h
#   │   ├── blake3_dispatch.c
#   │   ├── blake3_portable.c
#   │   └── ... (其他源文件)
#   ├── build/                # 构建目录
#   └── lib/                  # 输出库文件目录
#
# 🧩 可配置选项（通过命令行传入）：
#   -DCOMPILER_TARGET=x86|x64|arm32|arm64    # 目标平台架构
#   -DTARGET_OS=linux|windows                 # 目标操作系统
#   -DTARGET_TYPE=static|shared               # 目标类型：静态库 / 动态库
#   -DENABLE_TBB=ON|OFF                       # 是否启用TBB并行计算
#   -DENABLE_ASSEMBLY=ON|OFF                  # 是否启用汇编加速 (x64平台)
#   -DCMAKE_BUILD_TYPE=Release|Debug          # 构建类型
#
# ✅ 构建示例：
#   # Linux x64 静态库
#   mkdir build && cd build
#   cmake .. -DCOMPILER_TARGET=x64 -DTARGET_OS=linux -DTARGET_TYPE=static
#   make -j
#
#   # Windows ARM64 动态库 (使用MinGW交叉编译)
#   cmake .. -DCOMPILER_TARGET=arm64 -DTARGET_OS=windows -DTARGET_TYPE=shared
#
#   # Linux ARM32 静态库 (带NEON加速)
#   cmake .. -DCOMPILER_TARGET=arm32 -DTARGET_OS=linux -DTARGET_TYPE=static
####################################################################################################

cmake_minimum_required(VERSION 3.9)

# ---------------------------------------------------------------------------------------
# 1. 项目基础配置
# ---------------------------------------------------------------------------------------
project(libblake3 VERSION 1.8.3 LANGUAGES C ASM)

# 设置构建类型
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
endif()

# C语言标准设置 (BLAKE3需要C99)
set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS OFF)

# 平台和架构配置
set(TARGET_OS "linux" CACHE STRING "Target OS: linux, windows")
set(COMPILER_TARGET "x64" CACHE STRING "Target architecture: x86, x64, arm32, arm64")
set(TARGET_TYPE "static" CACHE STRING "Build target type: static | shared")
option(ENABLE_TBB "Enable Intel TBB parallelism" OFF)
option(ENABLE_ASSEMBLY "Enable assembly optimizations (x64 only)" ON)

# 根据目标类型设置库类型
if(TARGET_TYPE STREQUAL "shared")
    set(BUILD_SHARED_LIBS ON CACHE BOOL "Build shared library" FORCE)
else()
    set(BUILD_SHARED_LIBS OFF CACHE BOOL "Build static library" FORCE)
endif()

# ---------------------------------------------------------------------------------------
# 2. 编译器与交叉编译配置
# ---------------------------------------------------------------------------------------
# 设置目标系统处理器类型 (用于BLAKE3的SIMD检测)
if(COMPILER_TARGET STREQUAL "x64")
    set(CMAKE_SYSTEM_PROCESSOR "x86_64")
    set(BLAKE3_TARGET_ARCH "amd64")
elseif(COMPILER_TARGET STREQUAL "x86")
    set(CMAKE_SYSTEM_PROCESSOR "i686")
    set(BLAKE3_TARGET_ARCH "x86")
elseif(COMPILER_TARGET STREQUAL "arm64")
    set(CMAKE_SYSTEM_PROCESSOR "aarch64")
    set(BLAKE3_TARGET_ARCH "arm64")
elseif(COMPILER_TARGET STREQUAL "arm32")
    set(CMAKE_SYSTEM_PROCESSOR "arm")
    set(BLAKE3_TARGET_ARCH "arm")
endif()

message(STATUS "Building for ${TARGET_OS} ${COMPILER_TARGET} (${CMAKE_SYSTEM_PROCESSOR})")
message(STATUS "Library type: ${TARGET_TYPE}")
message(STATUS "TBB parallelism: ${ENABLE_TBB}")
message(STATUS "Assembly optimizations: ${ENABLE_ASSEMBLY}")

# 编译器配置
if(TARGET_OS STREQUAL "windows")
    # Windows平台配置
    if(CMAKE_HOST_SYSTEM_NAME MATCHES "Linux")
        # Linux下交叉编译Windows目标
        set(CMAKE_SYSTEM_NAME Windows)

        if(COMPILER_TARGET STREQUAL "x64")
            set(CMAKE_C_COMPILER x86_64-w64-mingw32-gcc)
            set(CMAKE_ASM_MASM_COMPILER x86_64-w64-mingw32-gcc)
        elseif(COMPILER_TARGET STREQUAL "x86")
            set(CMAKE_C_COMPILER i686-w64-mingw32-gcc)
            set(CMAKE_ASM_MASM_COMPILER i686-w64-mingw32-gcc)
        elseif(COMPILER_TARGET STREQUAL "arm64")
            set(CMAKE_C_COMPILER aarch64-w64-mingw32-gcc)
        endif()

        # Windows定义
        add_definitions(-DWIN32 -D_WIN32)

    else()
        # 原生Windows编译 (MSVC)
        add_definitions(-DWIN32 -D_WIN32 -D_CRT_SECURE_NO_WARNINGS)
    endif()

else()
    # Linux平台配置
    if(COMPILER_TARGET STREQUAL "arm64")
        set(CMAKE_C_COMPILER aarch64-linux-gnu-gcc)
    elseif(COMPILER_TARGET STREQUAL "arm32")
        set(CMAKE_C_COMPILER arm-linux-gnueabihf-gcc)
        # ARM32需要显式启用NEON
        add_compile_options(-mfpu=neon)
    elseif(COMPILER_TARGET STREQUAL "x64")
        set(CMAKE_C_COMPILER gcc)
    elseif(COMPILER_TARGET STREQUAL "x86")
        set(CMAKE_C_COMPILER gcc)
        add_compile_options(-m32)
        set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -m32")
        set(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -m32")
    endif()
endif()

# ---------------------------------------------------------------------------------------
# 3. BLAKE3专用配置
# ---------------------------------------------------------------------------------------
# SIMD加速配置
if(ENABLE_ASSEMBLY AND COMPILER_TARGET STREQUAL "x64")
    if(TARGET_OS STREQUAL "windows")
        set(BLAKE3_SIMD_TYPE "amd64-asm")
    else()
        set(BLAKE3_SIMD_TYPE "amd64-asm")
    endif()
elseif(COMPILER_TARGET STREQUAL "x64" OR COMPILER_TARGET STREQUAL "x86")
    set(BLAKE3_SIMD_TYPE "x86-intrinsics")
elseif(COMPILER_TARGET STREQUAL "arm64")
    set(BLAKE3_SIMD_TYPE "neon-intrinsics")
elseif(COMPILER_TARGET STREQUAL "arm32")
    set(BLAKE3_SIMD_TYPE "neon-intrinsics")
else()
    set(BLAKE3_SIMD_TYPE "none")
endif()

message(STATUS "BLAKE3 SIMD acceleration: ${BLAKE3_SIMD_TYPE}")

# 编译器标志设置
if(CMAKE_C_COMPILER_ID STREQUAL "GNU" OR CMAKE_C_COMPILER_ID STREQUAL "Clang" OR CMAKE_C_COMPILER_ID STREQUAL "AppleClang")
    # GCC/Clang编译器
    set(BLAKE3_CFLAGS_SSE2 "-msse2" CACHE STRING "Compiler flags for SSE2")
    set(BLAKE3_CFLAGS_SSE4_1 "-msse4.1" CACHE STRING "Compiler flags for SSE4.1")
    set(BLAKE3_CFLAGS_AVX2 "-mavx2" CACHE STRING "Compiler flags for AVX2")
    set(BLAKE3_CFLAGS_AVX512 "-mavx512f -mavx512vl" CACHE STRING "Compiler flags for AVX512")

    if(COMPILER_TARGET STREQUAL "arm32")
        set(BLAKE3_CFLAGS_NEON "-mfpu=neon" CACHE STRING "Compiler flags for NEON")
    endif()

elseif(MSVC)
    # MSVC编译器
    set(BLAKE3_CFLAGS_SSE2 "/arch:SSE2")
    set(BLAKE3_CFLAGS_SSE4_1 "/arch:AVX")  # MSVC没有专门的SSE4.1标志
    set(BLAKE3_CFLAGS_AVX2 "/arch:AVX2")
    set(BLAKE3_CFLAGS_AVX512 "/arch:AVX512")
endif()

# TBB配置
if(ENABLE_TBB)
    set(BLAKE3_USE_TBB ON CACHE BOOL "Enable TBB" FORCE)
    find_package(TBB QUIET)
    if(NOT TBB_FOUND)
        message(WARNING "TBB not found, disabling TBB support")
        set(ENABLE_TBB OFF)
        set(BLAKE3_USE_TBB OFF)
    endif()
else()
    set(BLAKE3_USE_TBB OFF)
endif()

# ---------------------------------------------------------------------------------------
# 4. 源文件配置
# ---------------------------------------------------------------------------------------
# BLAKE3核心源文件
set(BLAKE3_CORE_SOURCES
        blake3/blake3.c
        blake3/blake3_dispatch.c
        blake3/blake3_portable.c
        blake3/blake3.h
        blake3/blake3_impl.h
)

# 根据SIMD类型添加额外源文件
if(BLAKE3_SIMD_TYPE STREQUAL "amd64-asm" AND ENABLE_ASSEMBLY)
    if(TARGET_OS STREQUAL "windows")
        set(BLAKE3_ASM_SOURCES
                blake3/blake3_avx2_x86-64_windows_gnu.S
                blake3/blake3_avx512_x86-64_windows_gnu.S
                blake3/blake3_sse2_x86-64_windows_gnu.S
                blake3/blake3_sse41_x86-64_windows_gnu.S
        )
    else()
        set(BLAKE3_ASM_SOURCES
                blake3/blake3_avx2_x86-64_unix.S
                blake3/blake3_avx512_x86-64_unix.S
                blake3/blake3_sse2_x86-64_unix.S
                blake3/blake3_sse41_x86-64_unix.S
        )
    endif()
    list(APPEND BLAKE3_CORE_SOURCES ${BLAKE3_ASM_SOURCES})

elseif(BLAKE3_SIMD_TYPE STREQUAL "x86-intrinsics")
    set(BLAKE3_INTRINSIC_SOURCES
            blake3/blake3_avx2.c
            blake3/blake3_avx512.c
            blake3/blake3_sse2.c
            blake3/blake3_sse41.c
    )
    list(APPEND BLAKE3_CORE_SOURCES ${BLAKE3_INTRINSIC_SOURCES})

elseif(BLAKE3_SIMD_TYPE STREQUAL "neon-intrinsics")
    set(BLAKE3_NEON_SOURCES
            blake3/blake3_neon.c
    )
    list(APPEND BLAKE3_CORE_SOURCES ${BLAKE3_NEON_SOURCES})
endif()

# 添加TBB源文件
if(ENABLE_TBB AND BLAKE3_USE_TBB)
    list(APPEND BLAKE3_CORE_SOURCES blake3/blake3_tbb.cpp)
endif()

# 验证源文件存在
foreach(SRC_FILE ${BLAKE3_CORE_SOURCES})
    if(NOT EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/${SRC_FILE})
        message(WARNING "Source file not found: ${SRC_FILE}")
    endif()
endforeach()

# ---------------------------------------------------------------------------------------
# 5. 创建库目标
# ---------------------------------------------------------------------------------------
if(BUILD_SHARED_LIBS)
    add_library(blake3 SHARED ${BLAKE3_CORE_SOURCES})
    target_compile_definitions(blake3
            PUBLIC BLAKE3_DLL
            PRIVATE BLAKE3_DLL_EXPORTS
    )
else()
    add_library(blake3 STATIC ${BLAKE3_CORE_SOURCES})
endif()

# 设置目标属性 - 根据目标平台设置正确的库文件扩展名
if(TARGET_OS STREQUAL "windows")
    # Windows平台命名规则
    if(BUILD_SHARED_LIBS)
        # Windows动态库：blake3.dll
        set_target_properties(blake3 PROPERTIES
                VERSION ${PROJECT_VERSION}
                SOVERSION 0
                C_VISIBILITY_PRESET hidden
                OUTPUT_NAME "blake3"
                PREFIX ""  # 去掉默认的"lib"前缀
                SUFFIX ".dll"  # 使用.dll扩展名
                IMPORT_SUFFIX ".dll.a"  # 导入库后缀
        )

        # 同时生成导入库（.dll.a文件）和定义文件（.def）
        if(CMAKE_C_COMPILER_ID STREQUAL "GNU")
            # MinGW需要生成导入库
            set_target_properties(blake3 PROPERTIES
                    WINDOWS_EXPORT_ALL_SYMBOLS OFF
                    DEFINE_SYMBOL BLAKE3_DLL_EXPORTS
            )
        endif()
    else()
        # Windows静态库：libblake3.a (MinGW) 或 blake3.lib (MSVC)
        if(MSVC)
            set_target_properties(blake3 PROPERTIES
                    OUTPUT_NAME "blake3"
                    PREFIX ""
                    SUFFIX ".lib"
            )
        else()
            # MinGW静态库
            set_target_properties(blake3 PROPERTIES
                    OUTPUT_NAME "blake3"
                    PREFIX "lib"
                    SUFFIX ".a"
            )
        endif()
    endif()
else()
    # Linux平台命名规则
    if(BUILD_SHARED_LIBS)
        # Linux动态库：libblake3.so.1.8.3
        set_target_properties(blake3 PROPERTIES
                VERSION ${PROJECT_VERSION}
                SOVERSION 0
                C_VISIBILITY_PRESET hidden
                OUTPUT_NAME "blake3"
                PREFIX "lib"
                SUFFIX ".so"
        )
    else()
        # Linux静态库：libblake3.a
        set_target_properties(blake3 PROPERTIES
                OUTPUT_NAME "blake3"
                PREFIX "lib"
                SUFFIX ".a"
        )
    endif()
endif()

# 包含目录
target_include_directories(blake3 PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/blake3>
        $<INSTALL_INTERFACE:include>
)

# 编译选项
target_compile_features(blake3 PUBLIC c_std_99)

# SIMD文件编译标志
if(BLAKE3_SIMD_TYPE STREQUAL "x86-intrinsics")
    if(DEFINED BLAKE3_CFLAGS_AVX2)
        set_source_files_properties(blake3/blake3_avx2.c PROPERTIES COMPILE_FLAGS "${BLAKE3_CFLAGS_AVX2}")
    endif()
    if(DEFINED BLAKE3_CFLAGS_AVX512)
        set_source_files_properties(blake3/blake3_avx512.c PROPERTIES COMPILE_FLAGS "${BLAKE3_CFLAGS_AVX512}")
    endif()
    if(DEFINED BLAKE3_CFLAGS_SSE2)
        set_source_files_properties(blake3/blake3_sse2.c PROPERTIES COMPILE_FLAGS "${BLAKE3_CFLAGS_SSE2}")
    endif()
    if(DEFINED BLAKE3_CFLAGS_SSE4_1)
        set_source_files_properties(blake3/blake3_sse41.c PROPERTIES COMPILE_FLAGS "${BLAKE3_CFLAGS_SSE4_1}")
    endif()
endif()

if(BLAKE3_SIMD_TYPE STREQUAL "neon-intrinsics" AND DEFINED BLAKE3_CFLAGS_NEON)
    set_source_files_properties(blake3/blake3_neon.c PROPERTIES COMPILE_FLAGS "${BLAKE3_CFLAGS_NEON}")
endif()

# TBB链接
if(ENABLE_TBB AND BLAKE3_USE_TBB)
    target_link_libraries(blake3 PUBLIC TBB::tbb)
    target_compile_definitions(blake3 PUBLIC BLAKE3_USE_TBB)

    # C++编译选项 (TBB需要C++文件)
    if(CMAKE_C_COMPILER_ID STREQUAL "GNU" OR CMAKE_C_COMPILER_ID STREQUAL "Clang")
        target_compile_options(blake3 PRIVATE $<$<COMPILE_LANGUAGE:CXX>:-fno-exceptions -fno-rtti>)
    elseif(MSVC)
        target_compile_options(blake3 PRIVATE $<$<COMPILE_LANGUAGE:CXX>:/EHs-c- /GR->)
    endif()
endif()

# ---------------------------------------------------------------------------------------
# 6. 安装配置
# ---------------------------------------------------------------------------------------
# 安装头文件
install(FILES blake3/blake3.h DESTINATION include)

# 安装库文件
install(TARGETS blake3
        ARCHIVE DESTINATION lib
        LIBRARY DESTINATION lib
        RUNTIME DESTINATION bin
)

# 生成和安装配置文件
include(CMakePackageConfigHelpers)
configure_package_config_file(
        ${CMAKE_CURRENT_SOURCE_DIR}/cmake/blake3-config.cmake.in
        ${CMAKE_CURRENT_BINARY_DIR}/blake3-config.cmake
        INSTALL_DESTINATION lib/cmake/blake3
)

write_basic_package_version_file(
        ${CMAKE_CURRENT_BINARY_DIR}/blake3-config-version.cmake
        VERSION ${PROJECT_VERSION}
        COMPATIBILITY SameMajorVersion
)

install(FILES
        ${CMAKE_CURRENT_BINARY_DIR}/blake3-config.cmake
        ${CMAKE_CURRENT_BINARY_DIR}/blake3-config-version.cmake
        DESTINATION lib/cmake/blake3
)

# ---------------------------------------------------------------------------------------
# 7. 构建信息汇总
# ---------------------------------------------------------------------------------------
message(STATUS "------------------------------------------------")
message(STATUS " BLAKE3 Library Build Configuration")
message(STATUS "------------------------------------------------")
message(STATUS " Project Name:    ${PROJECT_NAME}")
message(STATUS " Version:         ${PROJECT_VERSION}")
message(STATUS " Target OS:       ${TARGET_OS}")
message(STATUS " Target Arch:     ${COMPILER_TARGET} (${CMAKE_SYSTEM_PROCESSOR})")
message(STATUS " Library Type:    ${TARGET_TYPE}")
message(STATUS " SIMD Acceleration: ${BLAKE3_SIMD_TYPE}")
message(STATUS " TBB Parallelism: ${ENABLE_TBB}")
message(STATUS " Build Type:      ${CMAKE_BUILD_TYPE}")
message(STATUS " Install Prefix:  ${CMAKE_INSTALL_PREFIX}")
message(STATUS " C Compiler:      ${CMAKE_C_COMPILER}")
message(STATUS " Source Files:    ${BLAKE3_CORE_SOURCES}")
message(STATUS "------------------------------------------------")