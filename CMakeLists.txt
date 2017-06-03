# cmake -DCMAKE_TOOLCHAIN_FILE=~/bfprefix/cmake/CMakeToolchain_VMM_40.txt -DDISABLE_WARNINGS=on ..

cmake_minimum_required(VERSION 3.6)
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
# Source Directory
# ------------------------------------------------------------------------------

if(NOT BFSOURCE_DIR)
    set(BFSOURCE_DIR "${CMAKE_INSTALL_PREFIX}/src")
else()
    if(NOT IS_ABSOLUTE ${BFSOURCE_DIR})
        set(BFSOURCE_DIR "${CMAKE_BINARY_DIR}/${BFSOURCE_DIR}")
    endif()
endif()

message("-- Source Directory: ${BFSOURCE_DIR}")

# ------------------------------------------------------------------------------
# Newlib
# ------------------------------------------------------------------------------

set(NEWLIB_CFLAGS
    "${CMAKE_C_FLAGS} -nostdinc-c"
)

list(APPEND NEWLIB_ARGS
    "--prefix=${CMAKE_BINARY_DIR}/newlib/prefix"
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
    TMP_DIR             ${CMAKE_BINARY_DIR}/newlib/tmp
    STAMP_DIR           ${CMAKE_BINARY_DIR}/newlib/stamp
    SOURCE_DIR          ${BFSOURCE_DIR}/newlib
    BINARY_DIR          ${CMAKE_BINARY_DIR}/newlib/build
    CONFIGURE_COMMAND   ${BFSOURCE_DIR}/newlib/configure "${NEWLIB_ARGS}" CFLAGS_FOR_TARGET=${NEWLIB_CFLAGS}
    BUILD_COMMAND       make
    INSTALL_COMMAND     make install
    LOG_CONFIGURE       1
    LOG_BUILD           1
    LOG_INSTALL         1
)

ExternalProject_Add_Step(
    newlib
    link
    COMMAND             eval "${CMAKE_C_COMPILER} -shared `find ${CMAKE_BINARY_DIR}/newlib/build/${BAREFLANK_TARGET}/newlib/libc/ -name \"*.o\" | xargs echo` -o ${CMAKE_BINARY_DIR}/newlib/libc.so"
    DEPENDEES           install
)

ExternalProject_Add_Step(
    newlib
    prefix_install
    COMMAND             ${CMAKE_COMMAND} -E copy_directory ${CMAKE_BINARY_DIR}/newlib/prefix/${BAREFLANK_TARGET}/ ${CMAKE_INSTALL_PREFIX}/sysroots/${BAREFLANK_TARGET}/
    COMMAND             ${CMAKE_COMMAND} -E copy ${CMAKE_BINARY_DIR}/newlib/libc.so ${CMAKE_INSTALL_PREFIX}/sysroots/${BAREFLANK_TARGET}/lib/
    DEPENDEES           link
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
    CMAKE_ARGS  ${BFUNWIND_CMAKE_ARGS}
    TMP_DIR     ${CMAKE_BINARY_DIR}/bfunwind/tmp
    STAMP_DIR   ${CMAKE_BINARY_DIR}/bfunwind/stamp
    SOURCE_DIR  ${BFSOURCE_DIR}/bfunwind
    BINARY_DIR  ${CMAKE_BINARY_DIR}/bfunwind/build
    DEPENDS     newlib
)

# ------------------------------------------------------------------------------
# Libcxxabi
# ------------------------------------------------------------------------------

list(APPEND LIBCXXABI_CMAKE_ARGS
    "-DCMAKE_INSTALL_PREFIX=${BAREFLANK_SYSROOT_PATH}"
    "-DLLVM_PATH=${BFSOURCE_DIR}/llvm"
    "-DLIBCXXABI_LIBCXX_PATH=${BFSOURCE_DIR}/libcxx"
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
    CMAKE_ARGS  ${LIBCXXABI_CMAKE_ARGS}
    TMP_DIR     ${CMAKE_BINARY_DIR}/libcxxabi/tmp
    STAMP_DIR   ${CMAKE_BINARY_DIR}/libcxxabi/stamp
    SOURCE_DIR  ${BFSOURCE_DIR}/libcxxabi
    BINARY_DIR  ${CMAKE_BINARY_DIR}/libcxxabi/build
    DEPENDS     bfunwind
)

# ------------------------------------------------------------------------------
# Libcxx
# ------------------------------------------------------------------------------

list(APPEND LIBCXX_CMAKE_ARGS
    "-DCMAKE_INSTALL_PREFIX=${BAREFLANK_SYSROOT_PATH}"
    "-DLLVM_PATH=${BFSOURCE_DIR}/llvm"
    "-DLIBCXX_CXX_ABI=libcxxabi"
    "-DLIBCXX_CXX_ABI_INCLUDE_PATHS=${BFSOURCE_DIR}/libcxxabi/include/"
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
    CMAKE_ARGS  ${LIBCXX_CMAKE_ARGS}
    TMP_DIR     ${CMAKE_BINARY_DIR}/libcxx/tmp
    STAMP_DIR   ${CMAKE_BINARY_DIR}/libcxx/stamp
    SOURCE_DIR  ${BFSOURCE_DIR}/libcxx
    BINARY_DIR  ${CMAKE_BINARY_DIR}/libcxx/build
    DEPENDS     bfunwind libcxxabi
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
    CMAKE_ARGS  ${BFSUPPORT_CMAKE_ARGS}
    TMP_DIR     ${CMAKE_BINARY_DIR}/bfsupport/tmp
    STAMP_DIR   ${CMAKE_BINARY_DIR}/bfsupport/stamp
    SOURCE_DIR  ${BFSOURCE_DIR}/bfsupport
    BINARY_DIR  ${CMAKE_BINARY_DIR}/bfsupport/build
    DEPENDS     newlib libcxxabi libcxx
)

# ------------------------------------------------------------------------------
# Clean
# ------------------------------------------------------------------------------

add_custom_target(super-clean
    COMMAND cmake --build . --target clean
    COMMAND cmake --build ${CMAKE_BINARY_DIR}/bfsupport/build --target clean
    COMMAND cmake --build ${CMAKE_BINARY_DIR}/bfunwind/build --target clean
    COMMAND cmake --build ${CMAKE_BINARY_DIR}/libcxx/build --target clean
    COMMAND cmake --build ${CMAKE_BINARY_DIR}/libcxxabi/build --target clean
    COMMAND make -C ${CMAKE_BINARY_DIR}/newlib/build clean > /dev/null 2>&1
)
