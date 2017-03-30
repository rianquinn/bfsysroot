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

include("${CMAKE_INSTALL_PREFIX}/cmake/CMakeGlobal_Project.txt")

# ------------------------------------------------------------------------------
# Newlib
# ------------------------------------------------------------------------------

set(NEWLIB_CFLAGS
    "${CMAKE_C_FLAGS} -fvisibility=default"
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
    GIT_REPOSITORY https://github.com/Bareflank/newlib.git
    GIT_TAG v1.2
    CONFIGURE_COMMAND ${CMAKE_BINARY_DIR}/newlib/src/newlib/configure "${NEWLIB_ARGS}" CFLAGS_FOR_TARGET=${NEWLIB_CFLAGS}
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
    GIT_REPOSITORY https://github.com/Bareflank/bfunwind.git
    CMAKE_ARGS ${BFUNWIND_CMAKE_ARGS}
    DEPENDS newlib
)

# ------------------------------------------------------------------------------
# Download LLVM
# ------------------------------------------------------------------------------

ExternalProject_Add(
    download_llvm
    PREFIX ${CMAKE_BINARY_DIR}/download_llvm
    DOWNLOAD_COMMAND git clone --depth 1 -b v1.2 https://github.com/Bareflank/llvm.git download_llvm
    CONFIGURE_COMMAND ""
    BUILD_COMMAND ""
    INSTALL_COMMAND ""
)

# ------------------------------------------------------------------------------
# Download Libcxxabi
# ------------------------------------------------------------------------------

ExternalProject_Add(
    download_libcxxabi
    PREFIX ${CMAKE_BINARY_DIR}/download_libcxxabi
    DOWNLOAD_COMMAND git clone --depth 1 -b v1.2 https://github.com/Bareflank/libcxxabi.git download_libcxxabi
    CONFIGURE_COMMAND ""
    BUILD_COMMAND ""
    INSTALL_COMMAND ""
)

# ------------------------------------------------------------------------------
# Download Libcxx
# ------------------------------------------------------------------------------

ExternalProject_Add(
    download_libcxx
    PREFIX ${CMAKE_BINARY_DIR}/download_libcxx
    DOWNLOAD_COMMAND git clone --depth 1 -b v1.2 https://github.com/Bareflank/libcxx.git download_libcxx
    CONFIGURE_COMMAND ""
    BUILD_COMMAND ""
    INSTALL_COMMAND ""
)

# ------------------------------------------------------------------------------
# Libcxxabi
# ------------------------------------------------------------------------------

list(APPEND LIBCXXABI_CMAKE_ARGS
    "-DCMAKE_INSTALL_PREFIX=${BAREFLANK_SYSROOT_PATH}"
    "-DLLVM_PATH=${CMAKE_BINARY_DIR}/download_llvm/src/download_llvm"
    "-DLIBCXXABI_LIBCXX_PATH=${CMAKE_BINARY_DIR}/download_libcxx/src/download_libcxx"
    "-DLIBCXXABI_SYSROOT=${BAREFLANK_SYSROOT_PATH}"
    "-DLIBCXXABI_HAS_PTHREAD_API=ON"
    "-DLLVM_ENABLE_LIBCXX=ON"
    "-DCMAKE_SYSTEM_NAME=${CMAKE_SYSTEM_NAME}"
    "-DCMAKE_C_COMPILER=${CMAKE_C_COMPILER}"
    "-DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}"
    "-DCMAKE_AR=${CMAKE_AR}"
    "-DCMAKE_C_FLAGS=${CMAKE_C_FLAGS} -DCOMPILING_LIBCXX"
    "-DCMAKE_CXX_FLAGS=${CMAKE_CXX_FLAGS} -DCOMPILING_LIBCXX"
)

ExternalProject_Add(
    libcxxabi
    PREFIX ${CMAKE_BINARY_DIR}/libcxxabi
    DOWNLOAD_COMMAND git clone --depth 1 -b v1.2 https://github.com/Bareflank/libcxxabi.git
    CMAKE_ARGS ${LIBCXXABI_CMAKE_ARGS}
    DEPENDS bfunwind download_llvm download_libcxx download_libcxxabi
)

# ------------------------------------------------------------------------------
# Libcxx
# ------------------------------------------------------------------------------

list(APPEND LIBCXX_CMAKE_ARGS
    "-DCMAKE_INSTALL_PREFIX=${BAREFLANK_SYSROOT_PATH}"
    "-DLLVM_PATH=${CMAKE_BINARY_DIR}/download_llvm/src/download_llvm"
    "-DLIBCXX_CXX_ABI=libcxxabi"
    "-DLIBCXX_CXX_ABI_INCLUDE_PATHS=${CMAKE_BINARY_DIR}/download_libcxxabi/src/download_libcxxabi/include/"
    "-DLIBCXX_SYSROOT=${BAREFLANK_SYSROOT_PATH}"
    "-DLIBCXX_HAS_PTHREAD_API=ON"
    "-DLIBCXX_ENABLE_EXPERIMENTAL_LIBRARY=OFF"
    "-DCMAKE_SYSTEM_NAME=${CMAKE_SYSTEM_NAME}"
    "-DCMAKE_C_COMPILER=${CMAKE_C_COMPILER}"
    "-DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}"
    "-DCMAKE_AR=${CMAKE_AR}"
    "-DCMAKE_C_FLAGS=${CMAKE_C_FLAGS} -DCOMPILING_LIBCXX"
    "-DCMAKE_CXX_FLAGS=${CMAKE_CXX_FLAGS} -DCOMPILING_LIBCXX"
)

ExternalProject_Add(
    libcxx
    PREFIX ${CMAKE_BINARY_DIR}/libcxx
    DOWNLOAD_COMMAND git clone --depth 1 -b v1.2 https://github.com/Bareflank/libcxx.git
    CMAKE_ARGS ${LIBCXX_CMAKE_ARGS}
    DEPENDS bfunwind download_llvm download_libcxx download_libcxxabi libcxxabi
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
    GIT_REPOSITORY https://github.com/Bareflank/bfsupport.git
    CMAKE_ARGS ${BFSUPPORT_CMAKE_ARGS}
    DEPENDS newlib libcxxabi libcxx
)
