# cmake -DCMAKE_TOOLCHAIN_FILE=~/bfprefix/cmake/CMakeToolchain_VMM_40.txt -DDISABLE_WARNINGS=on ..

cmake_minimum_required(VERSION 3.4)
project(bfsysroot C CXX)

if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
    if(NOT WIN32 STREQUAL "1")
        set(CMAKE_INSTALL_PREFIX "$ENV{HOME}/bfprefix" CACHE PATH "default install path" FORCE)
    else()
        set(CMAKE_INSTALL_PREFIX "$ENV{HOMEPATH}/bfprefix" CACHE PATH "default install path" FORCE)
    endif()
endif()

set(DISABLE_VISIBILITY_HIDDEN true)
include("${CMAKE_INSTALL_PREFIX}/cmake/CMakeGlobal_Project.txt")

# ------------------------------------------------------------------------------
# Newlib
# ------------------------------------------------------------------------------

set(NEWLIB_CFLAGS
    "${CMAKE_C_FLAGS} -nostdinc-c"
)

list(APPEND NEWLIB_ARGS
    "--prefix=${CMAKE_INSTALL_PREFIX}/sysroots/"
    "--target=${BAREFLANK_TARGET}"
    "--disable-libgloss"
    "--disable-multilib"
    "--disable-newlib-supplied-syscalls"
    "--enable-newlib-multithread"
    "--enable-newlib-iconv"
    "CC_FOR_TARGET=${CMAKE_C_COMPILER}"
    "CXX_FOR_TARGET=${CMAKE_CXX_COMPILER}"
)

ExternalProject_Add(
    newlib
    PREFIX ${CMAKE_BINARY_DIR}/newlib
    SOURCE_DIR ${CMAKE_INSTALL_PREFIX}/src/newlib
    CONFIGURE_COMMAND ${CMAKE_INSTALL_PREFIX}/src/newlib/configure "${NEWLIB_ARGS}" CFLAGS_FOR_TARGET=${NEWLIB_CFLAGS}
    BUILD_COMMAND make
    INSTALL_COMMAND make install
    TEST_COMMAND link_newlib.sh ${CMAKE_C_COMPILER} ${CMAKE_BINARY_DIR} ${BAREFLANK_SYSROOT_PATH}
    LOG_CONFIGURE 1
    LOG_BUILD 1
    LOG_INSTALL 1
)

# ------------------------------------------------------------------------------
# Unwind
# ------------------------------------------------------------------------------

list(APPEND BFUNWIND_CMAKE_ARGS
    "-DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}"
    "-DCMAKE_TOOLCHAIN_FILE=${CMAKE_TOOLCHAIN_FILE}"
)

ExternalProject_Add(
    bfunwind
    PREFIX ${CMAKE_BINARY_DIR}/bfunwind
    SOURCE_DIR ${CMAKE_INSTALL_PREFIX}/src/bfunwind
    CMAKE_ARGS ${BFUNWIND_CMAKE_ARGS}
    DEPENDS newlib
)

# ------------------------------------------------------------------------------
# Libcxxabi
# ------------------------------------------------------------------------------

list(APPEND LIBCXXABI_CMAKE_ARGS
    "-DCMAKE_INSTALL_PREFIX=${BAREFLANK_SYSROOT_PATH}"
    "-DLLVM_PATH=${CMAKE_INSTALL_PREFIX}/src/llvm"
    "-DLIBCXXABI_LIBCXX_PATH=${CMAKE_INSTALL_PREFIX}/src/libcxx"
    "-DLIBCXXABI_SYSROOT=${BAREFLANK_SYSROOT_PATH}"
    "-DLIBCXXABI_HAS_PTHREAD_API=ON"
    "-DLLVM_ENABLE_LIBCXX=ON"
    "-DCMAKE_SYSTEM_NAME=${CMAKE_SYSTEM_NAME}"
    "-DCMAKE_C_COMPILER=${CMAKE_C_COMPILER}"
    "-DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}"
    "-DCMAKE_AR=${CMAKE_AR}"
    "-DCMAKE_C_FLAGS=${CMAKE_C_FLAGS}"
    "-DCMAKE_CXX_FLAGS=${CMAKE_CXX_FLAGS} -nostdinc-c++"
)

ExternalProject_Add(
    libcxxabi
    PREFIX ${CMAKE_BINARY_DIR}/libcxxabi
    SOURCE_DIR ${CMAKE_INSTALL_PREFIX}/src/libcxxabi
    CMAKE_ARGS ${LIBCXXABI_CMAKE_ARGS}
    DEPENDS bfunwind
)

# ------------------------------------------------------------------------------
# Libcxx
# ------------------------------------------------------------------------------

list(APPEND LIBCXX_CMAKE_ARGS
    "-DCMAKE_INSTALL_PREFIX=${BAREFLANK_SYSROOT_PATH}"
    "-DLLVM_PATH=${CMAKE_INSTALL_PREFIX}/src/llvm"
    "-DLIBCXX_CXX_ABI=libcxxabi"
    "-DLIBCXX_CXX_ABI_INCLUDE_PATHS=${CMAKE_INSTALL_PREFIX}/src/libcxxabi/include/"
    "-DLIBCXX_SYSROOT=${BAREFLANK_SYSROOT_PATH}"
    "-DLIBCXX_HAS_PTHREAD_API=ON"
    "-DLIBCXX_ENABLE_EXPERIMENTAL_LIBRARY=OFF"
    "-DCMAKE_SYSTEM_NAME=${CMAKE_SYSTEM_NAME}"
    "-DCMAKE_C_COMPILER=${CMAKE_C_COMPILER}"
    "-DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}"
    "-DCMAKE_AR=${CMAKE_AR}"
    "-DCMAKE_C_FLAGS=${CMAKE_C_FLAGS}"
    "-DCMAKE_CXX_FLAGS=${CMAKE_CXX_FLAGS} -nostdinc-c++"
)

ExternalProject_Add(
    libcxx
    PREFIX ${CMAKE_BINARY_DIR}/libcxx
    SOURCE_DIR ${CMAKE_INSTALL_PREFIX}/src/libcxx
    CMAKE_ARGS ${LIBCXX_CMAKE_ARGS}
    DEPENDS bfunwind libcxxabi
)

# ------------------------------------------------------------------------------
# Support
# ------------------------------------------------------------------------------

list(APPEND BFSUPPORT_CMAKE_ARGS
    "-DCMAKE_INSTALL_PREFIX=${CMAKE_INSTALL_PREFIX}"
    "-DCMAKE_TOOLCHAIN_FILE=${CMAKE_TOOLCHAIN_FILE}"
)

ExternalProject_Add(
    bfsupport
    PREFIX ${CMAKE_BINARY_DIR}/bfsupport
    SOURCE_DIR ${CMAKE_INSTALL_PREFIX}/src/bfsupport
    CMAKE_ARGS ${BFSUPPORT_CMAKE_ARGS}
    DEPENDS newlib libcxxabi libcxx
)
