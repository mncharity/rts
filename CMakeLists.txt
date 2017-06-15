cmake_minimum_required(VERSION 3.2.2)
project(RTS VERSION 0.1 LANGUAGES CXX)
set(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${CMAKE_SOURCE_DIR}/etc/cmake/Modules)
string(TOLOWER ${PROJECT_NAME} LOWER_PROJECT_NAME)
set(RTS_PROJECT_BRIEF "rts")

# Hook for VSCode CMakeTools
include(CMakeToolsHelpers OPTIONAL)

include(CTest)

if(MSVC)
  message("libm not required")
else()
	set(MATH_LIBRARIES "m")
  message("libm required")
endif()

# use foo.o rather than foo.cpp.o
set(CMAKE_CXX_OUTPUT_EXTENSION_REPLACE 1)

#option(ENABLE_COVERAGE "COVERAGE" ON)
#if(ENABLE_COVERAGE)
  #if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
    #add_compile_options(-fprofile-arcs -ftest-coverage -O0)
    #set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fprofile-arcs -ftest-coverage")
    #set(USE_COVERAGE ON)
  #endif()
#endif()

option(ENABLE_DOCS "DOCS" ON)
if(ENABLE_DOCS)
  find_package(Doxygen)
  if (DOXYGEN_FOUND)
    configure_file(${CMAKE_SOURCE_DIR}/etc/Doxyfile.in ${CMAKE_BINARY_DIR}/etc/Doxyfile @ONLY)
    add_custom_target(doc ALL ${DOXYGEN_EXECUTABLE} ${CMAKE_BINARY_DIR}/etc/Doxyfile
      WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
      COMMENT "Generating API documentation with Doxygen"
      VERBATIM)
  endif(DOXYGEN_FOUND)
else()
  message("Documentation disabled")
endif()

include(CheckCXXCompilerFlag)

# Boost
if(MSVC)
  #CHECK_CXX_COMPILER_FLAG("/MP" CXX_SUPPORTS_MP) 
  #if (CXX_SUPPORTS_MP)
    add_compile_options(/MP) # parallel builds, also needed to avoid write conflicts during parallel testing
  #endif()
endif()

# link with lld if available for speed
option(ENABLE_LLD "LLD" ON)
if (ENABLE_LLD)
  execute_process(COMMAND ${CMAKE_CXX_COMPILER} -fuse-ld=lld -Wl,--version ERROR_QUIET OUTPUT_VARIABLE LLD_VERSION)
  if(LLD_VERSION MATCHES "LLD")
    SET(LLD_AVAILABLE ON)
    MESSAGE(STATUS "lld linker available")
  endif()
endif()

# link with gold if available for speed
option(ENABLE_GOLD "GOLD" ON)
if (ENABLE_LLD)
  execute_process(COMMAND ${CMAKE_CXX_COMPILER} -fuse-ld=gold -Wl,--version ERROR_QUIET OUTPUT_VARIABLE GOLD_VERSION)
  if(GOLD_VERSION MATCHES "GNU gold")
    SET(GOLD_AVAILABLE ON)
    MESSAGE(STATUS "gold linker available")
  endif()
endif()

# favor lld if both are available
if(ENABLE_LLD AND LLD_AVAILABLE)
  MESSAGE(STATUS "Using lld linker")
  set (CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fuse-ld=lld")
  set (CMAKE_MODULE_LINKER_FLAGS "${CMAKE_MODULE_LINKER_FLAGS} -fuse-ld=lld")
  set (CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -fuse-ld=lld")
elseif(ENABLE_GOLD AND GOLD_AVAILABLE)
  MESSAGE(STATUS "Using gold linker")
  set (CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fuse-ld=gold")
  set (CMAKE_MODULE_LINKER_FLAGS "${CMAKE_MODULE_LINKER_FLAGS} -fuse-ld=gold")
  set (CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -fuse-ld=gold")
endif()

# fetch into build dir, we can't cache it in the submodule dir
SET(BOOST_DOWNLOAD_TO_BINARY_DIR ON CACHE BOOL "Download into binary directory for caching")
add_subdirectory("opt/boost-cmake")

CHECK_CXX_COMPILER_FLAG("-std=c++11" CXX_SUPPORTS_CXX11)
# for eventual fallback?

# c++14
CHECK_CXX_COMPILER_FLAG("-std=c++14" CXX_SUPPORTS_CXX14)
if(CXX_SUPPORTS_CXX14)
  if(MSVC)
    add_compile_options("-std=c++14")
  else()
    add_compile_options("$<$<COMPILE_LANGUAGE:CXX>:-std=c++14>")
  endif()
else()
  CHECK_CXX_COMPILER_FLAG("-std=c++1y" CXX_SUPPORTS_CXX1Y)
  if(CXX_SUPPORTS_CXX1Y)
    if(MSVC)
      add_compile_options("-std=c++1y")
    else()
      add_compile_options("$<$<COMPILE_LANGUAGE:CXX>:-std=c++1y>")
    endif()
  else()
    message(STATUS "The compiler ${CMAKE_CXX_COMPILER} has no C++14 support. Please use a different C++ compiler.")
  endif()
endif()

# -march
# -DMARCH=native  Permit any instructions supported by the local cpu.
# -DMARCH=OFF     Don't set -march.
# Doesn't affect MSVC.
set(MARCH "native" CACHE STRING "-march=MARCH {OFF,native,...}")
if(MARCH)
  if(NOT MSVC)
    CHECK_CXX_COMPILER_FLAG("-march=native" CXX_SUPPORTS_MARCH)
    if(CXX_SUPPORTS_MARCH)
      add_compile_options("-march=${MARCH}")
    endif()
  endif()
endif()

# for development
set(COMPILE_OPTIONS "" CACHE STRING "Add compile options")
if(COMPILE_OPTIONS)
  add_compile_options("${COMPILE_OPTIONS}")
endif()

# AVX
set(ENABLE_AVX "" CACHE STRING "{-mavx,-mno-avx,default} is {ON,OFF,\"\"}")
set_property(CACHE ENABLE_AVX PROPERTY STRINGS ON OFF "")
if(ENABLE_AVX STREQUAL "")
  # MSVC doesn't have a "use AVX if available", so we default to requiring it.
  if(MSVC)
    add_compile_options("/arch:AVX")
  endif()
elseif(ENABLE_AVX)
  if(MSVC)
    add_compile_options("/arch:AVX")
  endif()
  if(NOT MSVC)
    CHECK_CXX_COMPILER_FLAG("-mavx" CXX_SUPPORTS_MAVX)
    if(CXX_SUPPORTS_MAVX)
      add_compile_options("-mavx")
    endif()
  endif()
else()
  # MSVC disables AVX by default.
  if(NOT MSVC)
    CHECK_CXX_COMPILER_FLAG("-mno-avx" CXX_SUPPORTS_MNOAVX)
    if(CXX_SUPPORTS_MNOAVX)
      add_compile_options("-mno-avx")
    endif()
  endif()
endif()

# AVX2
set(ENABLE_AVX2 "" CACHE STRING "{-mavx2,-mno-avx2,default} is {ON,OFF,\"\"}")
set_property(CACHE ENABLE_AVX2 PROPERTY STRINGS ON OFF "")
if(ENABLE_AVX2 STREQUAL "")
  # MSVC doesn't have a "use AVX2 if available", so we default to requiring it.
  if(MSVC)
    add_compile_options("/arch:AVX2")
  endif()
elseif(ENABLE_AVX2)
  if(MSVC)
    add_compile_options("/arch:AVX2")
  endif()
  if(NOT MSVC)
    CHECK_CXX_COMPILER_FLAG("-mavx2" CXX_SUPPORTS_MAVX2)
    if(CXX_SUPPORTS_MAVX2)
      add_compile_options("-mavx2")
    endif()
  endif()
else()
  # MSVC disables AVX2 by default.
  if(NOT MSVC)
    CHECK_CXX_COMPILER_FLAG("-mno-avx2" CXX_SUPPORTS_MNOAVX2)
    if(CXX_SUPPORTS_MNOAVX2)
      add_compile_options("-mno-avx2")
    endif()
  endif()
endif()

# AVX512F
set(ENABLE_AVX512F "" CACHE STRING "{-mavx512f,-mno-avx512f,default} is {ON,OFF,\"\"}")
set_property(CACHE ENABLE_AVX512F PROPERTY STRINGS ON OFF "")
if(ENABLE_AVX512F STREQUAL "")
elseif(ENABLE_AVX512F)
  if(NOT MSVC)
    CHECK_CXX_COMPILER_FLAG("-mavx512f" CXX_SUPPORTS_MAVX512F)
    if(CXX_SUPPORTS_MAVX512F)
      add_compile_options("-mavx512f")
    endif()
  endif()
else()
  if(NOT MSVC)
    CHECK_CXX_COMPILER_FLAG("-mno-avx512f" CXX_SUPPORTS_MNOAVX512F)
    if(CXX_SUPPORTS_MNOAVX512F)
      add_compile_options("-mno-avx512f")
    endif()
  endif()
endif()

# FMA
set(ENABLE_FMA "" CACHE STRING "{-mfma,-mno-fma,default} is {ON,OFF,\"\"}")
set_property(CACHE ENABLE_FMA PROPERTY STRINGS ON OFF "")
if(ENABLE_FMA STREQUAL "")
elseif(ENABLE_FMA)
  if(NOT MSVC)
    CHECK_CXX_COMPILER_FLAG("-mfma" CXX_SUPPORTS_MFMA)
    if(CXX_SUPPORTS_MFMA)
      add_compile_options("-mfma")
    endif()
  endif()
else()
  if(NOT MSVC)
    CHECK_CXX_COMPILER_FLAG("-mno-fma" CXX_SUPPORTS_MNOFMA)
    if(CXX_SUPPORTS_MNOFMA)
      add_compile_options("-mno-fma")
    endif()
  endif()
endif()

# RTM
set(ENABLE_RTM "" CACHE STRING "{-mrtm,-mno-rtm,default} is {ON,OFF,\"\"}")
set_property(CACHE ENABLE_RTM PROPERTY STRINGS ON OFF "")
if(ENABLE_RTM STREQUAL "")
elseif(ENABLE_RTM)
  if(NOT MSVC)
    CHECK_CXX_COMPILER_FLAG("-mrtm" CXX_SUPPORTS_MRTM)
    if(CXX_SUPPORTS_MRTM)
      add_compile_options("-mrtm")
    endif()
  endif()
else()
  if(NOT MSVC)
    CHECK_CXX_COMPILER_FLAG("-mno-rtm" CXX_SUPPORTS_MNORTM)
    if(CXX_SUPPORTS_MNORTM)
      add_compile_options("-mno-rtm")
    endif()
  endif()
endif()

include_directories(${CMAKE_SOURCE_DIR} ${Boost_INCLUDE_DIRS} ${Event_INCLUDE_DIRS})

# build the library
add_subdirectory(src)

# build the test suite
if(BUILD_TESTING)
  add_subdirectory(t)
endif()
