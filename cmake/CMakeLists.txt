cmake_minimum_required(VERSION 3.18)
project(aapt2 LANGUAGES C CXX)

# ---------------------------------------------------------------------------
# Build configuration
# ---------------------------------------------------------------------------
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_C_STANDARD 11)

# AOSP root must be passed as -DAOSP_ROOT=<path>
set(AOSP_ROOT "" CACHE PATH "Path to the AOSP source root (repo workspace)")
if(NOT AOSP_ROOT OR NOT EXISTS "${AOSP_ROOT}")
  message(FATAL_ERROR
    "AOSP_ROOT must be set to a valid directory.\n"
    "Pass -DAOSP_ROOT=<path> to cmake.")
endif()

# Convenience paths (mirrors repo manifest project paths)
set(FWK_BASE   "${AOSP_ROOT}/frameworks/base")
set(SYS_CORE   "${AOSP_ROOT}/system/core")
set(SYS_BASE   "${AOSP_ROOT}/system/libbase")
set(SYS_LOG    "${AOSP_ROOT}/system/logging")
set(EXT_PROTO  "${AOSP_ROOT}/external/protobuf")
set(EXT_PNG    "${AOSP_ROOT}/external/libpng")
set(EXT_EXPAT  "${AOSP_ROOT}/external/expat")
set(EXT_ZLIB   "${AOSP_ROOT}/external/zlib")

# Proto-generated output directory
set(PROTO_OUT "${CMAKE_BINARY_DIR}/proto_gen")
file(MAKE_DIRECTORY "${PROTO_OUT}")

# Host protoc binary (must be available on PATH or set via -DPROTOC=...)
find_program(PROTOC protoc REQUIRED
  DOC "Path to the host protoc binary (must match AOSP external/protobuf version)")

# ---------------------------------------------------------------------------
# Suppress warnings from third-party AOSP code
# ---------------------------------------------------------------------------
add_compile_options(-w)

# Common definitions expected by AOSP source
add_compile_definitions(
  _GNU_SOURCE
  _FILE_OFFSET_BITS=64
  ANDROID_UTILS_REF_BASE_DISABLE_IMPLICIT_CONSTRUCTION
)

# ---------------------------------------------------------------------------
# zlib — prefer NDK-provided libz; fall back to AOSP external/zlib
# ---------------------------------------------------------------------------
find_library(LIBZ z)
if(LIBZ)
  add_library(z_import INTERFACE)
  target_link_libraries(z_import INTERFACE "${LIBZ}")
  set(ZLIB_TARGET z_import)
else()
  file(GLOB ZLIB_SRCS "${EXT_ZLIB}/src/*.c")
  list(FILTER ZLIB_SRCS EXCLUDE REGEX ".*gz(lib|open|read|write|close)\\.c$")
  add_library(z_static STATIC ${ZLIB_SRCS})
  target_include_directories(z_static PUBLIC "${EXT_ZLIB}/src")
  set(ZLIB_TARGET z_static)
endif()

# ---------------------------------------------------------------------------
# libpng
# ---------------------------------------------------------------------------
file(GLOB LIBPNG_SRCS "${EXT_PNG}/*.c")
list(FILTER LIBPNG_SRCS EXCLUDE REGEX ".*(pngtest|example|pngvalid)\\.c$")
add_library(png_static STATIC ${LIBPNG_SRCS})
target_include_directories(png_static PUBLIC "${EXT_PNG}")
target_compile_definitions(png_static PRIVATE
  PNG_ARM_NEON_IMPLEMENTATION=0
  PNG_NO_MMX_CODE)
target_link_libraries(png_static PRIVATE ${ZLIB_TARGET})

# ---------------------------------------------------------------------------
# libexpat
# ---------------------------------------------------------------------------
set(EXPAT_SRCS
  "${EXT_EXPAT}/lib/xmlparse.c"
  "${EXT_EXPAT}/lib/xmlrole.c"
  "${EXT_EXPAT}/lib/xmltok.c"
)
add_library(expat_static STATIC ${EXPAT_SRCS})
target_include_directories(expat_static PUBLIC "${EXT_EXPAT}/lib")
target_compile_definitions(expat_static PRIVATE
  HAVE_EXPAT_CONFIG_H=0
  XML_CONTEXT_BYTES=1024
  XML_DTD
  XML_NS
  BYTEORDER=1234)

# ---------------------------------------------------------------------------
# libprotobuf-cpp-full (from AOSP external/protobuf)
# aapt2 links against libprotobuf-cpp-full (with reflection/descriptor support)
# ---------------------------------------------------------------------------
file(GLOB_RECURSE PROTOBUF_SRCS "${EXT_PROTO}/src/google/protobuf/*.cc")
# Exclude the protoc compiler sources and tests
list(FILTER PROTOBUF_SRCS EXCLUDE REGEX ".*/compiler/.*\\.cc$")
list(FILTER PROTOBUF_SRCS EXCLUDE REGEX ".*_test(ing)?\\.cc$")
list(FILTER PROTOBUF_SRCS EXCLUDE REGEX ".*benchmark.*\\.cc$")
list(FILTER PROTOBUF_SRCS EXCLUDE REGEX ".*main\\.cc$")
list(FILTER PROTOBUF_SRCS EXCLUDE REGEX ".*/test_util.*\\.cc$")
add_library(protobuf_static STATIC ${PROTOBUF_SRCS})
target_include_directories(protobuf_static PUBLIC
  "${EXT_PROTO}/src"
  "${EXT_PROTO}/android")
target_compile_definitions(protobuf_static PRIVATE
  GOOGLE_PROTOBUF_NO_RTTI
  HAVE_PTHREAD)

# ---------------------------------------------------------------------------
# liblog (system/logging/liblog)
# ---------------------------------------------------------------------------
file(GLOB LIBLOG_SRCS "${SYS_LOG}/liblog/*.cpp")
list(FILTER LIBLOG_SRCS EXCLUDE REGEX ".*_test\\.cpp$")
add_library(log_static STATIC ${LIBLOG_SRCS})
target_include_directories(log_static PUBLIC
  "${SYS_LOG}/liblog/include")
target_compile_definitions(log_static PRIVATE
  LIBLOG_LOG_TAG="liblog"
  SNET_EVENT_LOG_TAG=1397638484)

# ---------------------------------------------------------------------------
# libbase (system/libbase)
# ---------------------------------------------------------------------------
file(GLOB LIBBASE_SRCS "${SYS_BASE}/*.cpp")
list(FILTER LIBBASE_SRCS EXCLUDE REGEX ".*_test\\.cpp$")
list(FILTER LIBBASE_SRCS EXCLUDE REGEX ".*_benchmark\\.cpp$")
# posix_strerror_r.cpp may not exist in all revisions; guard
list(FILTER LIBBASE_SRCS EXCLUDE REGEX ".*_windows.*\\.cpp$")
add_library(base_static STATIC ${LIBBASE_SRCS})
target_include_directories(base_static PUBLIC
  "${SYS_BASE}/include")
target_link_libraries(base_static PRIVATE log_static)

# ---------------------------------------------------------------------------
# libcutils (system/core/libcutils)
# ---------------------------------------------------------------------------
file(GLOB LIBCUTILS_SRCS
  "${SYS_CORE}/libcutils/*.cpp"
  "${SYS_CORE}/libcutils/*.c")
list(FILTER LIBCUTILS_SRCS EXCLUDE REGEX ".*_test\\.cpp$")
list(FILTER LIBCUTILS_SRCS EXCLUDE REGEX ".*_windows.*")
add_library(cutils_static STATIC ${LIBCUTILS_SRCS})
target_include_directories(cutils_static PUBLIC
  "${SYS_CORE}/libcutils/include")
target_link_libraries(cutils_static PRIVATE log_static base_static)

# ---------------------------------------------------------------------------
# libutils (system/core/libutils)
# ---------------------------------------------------------------------------
set(LIBUTILS_SRCS
  "${SYS_CORE}/libutils/Errors.cpp"
  "${SYS_CORE}/libutils/FileMap.cpp"
  "${SYS_CORE}/libutils/JenkinsHash.cpp"
  "${SYS_CORE}/libutils/NativeHandle.cpp"
  "${SYS_CORE}/libutils/Printer.cpp"
  "${SYS_CORE}/libutils/RefBase.cpp"
  "${SYS_CORE}/libutils/SharedBuffer.cpp"
  "${SYS_CORE}/libutils/StopWatch.cpp"
  "${SYS_CORE}/libutils/String8.cpp"
  "${SYS_CORE}/libutils/String16.cpp"
  "${SYS_CORE}/libutils/StrongPointer.cpp"
  "${SYS_CORE}/libutils/SystemClock.cpp"
  "${SYS_CORE}/libutils/Threads.cpp"
  "${SYS_CORE}/libutils/Timers.cpp"
  "${SYS_CORE}/libutils/Tokenizer.cpp"
  "${SYS_CORE}/libutils/Unicode.cpp"
  "${SYS_CORE}/libutils/VectorImpl.cpp"
  "${SYS_CORE}/libutils/misc.cpp"
)
# Filter to only existing files (some may not exist in all AOSP revisions)
set(_UTILS_SRCS_FILTERED "")
foreach(f IN LISTS LIBUTILS_SRCS)
  if(EXISTS "${f}")
    list(APPEND _UTILS_SRCS_FILTERED "${f}")
  endif()
endforeach()
add_library(utils_static STATIC ${_UTILS_SRCS_FILTERED})
target_include_directories(utils_static PUBLIC
  "${SYS_CORE}/libutils/include")
target_link_libraries(utils_static PRIVATE cutils_static log_static base_static)

# ---------------------------------------------------------------------------
# libziparchive (system/core/libziparchive)
# ---------------------------------------------------------------------------
set(ZIPARCHIVE_SRCS
  "${SYS_CORE}/libziparchive/zip_archive.cc"
  "${SYS_CORE}/libziparchive/zip_archive_stream_entry.cc"
  "${SYS_CORE}/libziparchive/zip_writer.cc"
)
# zip_cd_entry_map.cc and zip_error.cpp exist in newer AOSP revisions
foreach(f
    "${SYS_CORE}/libziparchive/zip_cd_entry_map.cc"
    "${SYS_CORE}/libziparchive/zip_error.cpp")
  if(EXISTS "${f}")
    list(APPEND ZIPARCHIVE_SRCS "${f}")
  endif()
endforeach()
add_library(ziparchive_static STATIC ${ZIPARCHIVE_SRCS})
target_include_directories(ziparchive_static PUBLIC
  "${SYS_CORE}/libziparchive/include")
target_include_directories(ziparchive_static PRIVATE
  "${SYS_CORE}/libziparchive/include_private")
target_link_libraries(ziparchive_static PRIVATE
  base_static log_static ${ZLIB_TARGET})

# ---------------------------------------------------------------------------
# libandroidfw (frameworks/base/libs/androidfw)
# ---------------------------------------------------------------------------
file(GLOB ANDROIDFW_SRCS "${FWK_BASE}/libs/androidfw/*.cpp")
list(FILTER ANDROIDFW_SRCS EXCLUDE REGEX ".*_test\\.cpp$")
list(FILTER ANDROIDFW_SRCS EXCLUDE REGEX ".*[Bb]enchmark\\.cpp$")

# Determine the include path (differs slightly between AOSP revisions)
set(ANDROIDFW_INCLUDE_DIR "${FWK_BASE}/libs/androidfw/include")
if(NOT EXISTS "${ANDROIDFW_INCLUDE_DIR}")
  set(ANDROIDFW_INCLUDE_DIR "${FWK_BASE}/libs/androidfw/include/androidfw")
endif()

add_library(androidfw_static STATIC ${ANDROIDFW_SRCS})
target_include_directories(androidfw_static PUBLIC
  "${ANDROIDFW_INCLUDE_DIR}"
  "${FWK_BASE}/libs/androidfw/include")
target_link_libraries(androidfw_static PRIVATE
  utils_static cutils_static base_static log_static
  ziparchive_static expat_static png_static protobuf_static
  ${ZLIB_TARGET})

# ---------------------------------------------------------------------------
# Protobuf generation for aapt2's .proto files
# ---------------------------------------------------------------------------
set(AAPT2_PROTO_DIR "${FWK_BASE}/tools/aapt2/proto")

# aapt2 also imports protos from the framework's core/proto directory
set(FWK_PROTO_DIR "${FWK_BASE}/core/proto")

file(GLOB AAPT2_PROTO_FILES "${AAPT2_PROTO_DIR}/*.proto")

set(PROTO_GEN_SRCS "")
foreach(PROTO_FILE IN LISTS AAPT2_PROTO_FILES)
  get_filename_component(PROTO_NAME "${PROTO_FILE}" NAME_WE)
  set(PROTO_CC "${PROTO_OUT}/${PROTO_NAME}.pb.cc")
  set(PROTO_H  "${PROTO_OUT}/${PROTO_NAME}.pb.h")
  add_custom_command(
    OUTPUT "${PROTO_CC}" "${PROTO_H}"
    COMMAND "${PROTOC}"
      --proto_path="${AAPT2_PROTO_DIR}"
      --proto_path="${FWK_PROTO_DIR}"
      --cpp_out="${PROTO_OUT}"
      "${PROTO_FILE}"
    DEPENDS "${PROTO_FILE}"
    COMMENT "Compiling protobuf: ${PROTO_NAME}.proto"
    VERBATIM)
  list(APPEND PROTO_GEN_SRCS "${PROTO_CC}")
endforeach()

# ---------------------------------------------------------------------------
# libaapt2_static — core aapt2 library (everything except cmd/ and Main.cpp)
# ---------------------------------------------------------------------------
file(GLOB_RECURSE LIBAAPT2_SRCS "${FWK_BASE}/tools/aapt2/*.cpp")
list(FILTER LIBAAPT2_SRCS EXCLUDE REGEX ".*_test\\.cpp$")
list(FILTER LIBAAPT2_SRCS EXCLUDE REGEX ".*/cmd/.*\\.cpp$")
list(FILTER LIBAAPT2_SRCS EXCLUDE REGEX ".*/Main\\.cpp$")
list(FILTER LIBAAPT2_SRCS EXCLUDE REGEX ".*[Bb]enchmark\\.cpp$")

add_library(libaapt2_static STATIC
  ${LIBAAPT2_SRCS}
  ${PROTO_GEN_SRCS})

target_include_directories(libaapt2_static PUBLIC
  "${FWK_BASE}/tools/aapt2"
  "${PROTO_OUT}")

target_link_libraries(libaapt2_static PUBLIC
  androidfw_static
  base_static
  utils_static
  cutils_static
  log_static
  ziparchive_static
  expat_static
  png_static
  protobuf_static
  ${ZLIB_TARGET})

# ---------------------------------------------------------------------------
# aapt2 executable
# ---------------------------------------------------------------------------
file(GLOB AAPT2_CMD_SRCS "${FWK_BASE}/tools/aapt2/cmd/*.cpp")
list(FILTER AAPT2_CMD_SRCS EXCLUDE REGEX ".*_test\\.cpp$")

add_executable(aapt2
  "${FWK_BASE}/tools/aapt2/Main.cpp"
  ${AAPT2_CMD_SRCS})

target_include_directories(aapt2 PRIVATE
  "${FWK_BASE}/tools/aapt2"
  "${PROTO_OUT}")

target_link_libraries(aapt2 PRIVATE libaapt2_static)
