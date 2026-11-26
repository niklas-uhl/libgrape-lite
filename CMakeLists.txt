# katric branch: libgrape-lite as a library for katric's competitor adapter (grape::grape-lite), plus the LCC app's
# headers (examples/analytical_apps). No executables, tests, docs, CUDA, gflags, jemalloc or rdkafka.
#
# Upstream sets everything globally (CMAKE_CXX_FLAGS, add_definitions, include_directories). Here it is all on the
# target. Most of GRAPE is header-only and compiled in the consumer's translation unit, so the settings that change
# its code (SIMD intersection and sort, -mavx2, -march=native, hugepages) are INTERFACE: they reach the one adapter
# that links this target, nothing else.
cmake_minimum_required(VERSION 3.15)
project(libgrape-lite VERSION 0.3.5 LANGUAGES C CXX)

include(FetchContent)

# The published Graphalytics runs: cmake -DUSE_HUGEPAGES=ON -DUSE_SIMD=ON, runtime flag --opt.
if(CMAKE_SYSTEM_PROCESSOR MATCHES "x86_64|AMD64")
  set(GRAPE_SIMD_DEFAULT ON)
else()
  set(GRAPE_SIMD_DEFAULT OFF) # the SIMD intersection uses SSE4.2 / AVX2
endif()
option(GRAPE_USE_SIMD "SIMD intersection and sort, as in GRAPE's Graphalytics runs (x86-64 only)" ${GRAPE_SIMD_DEFAULT})
option(GRAPE_USE_HUGEPAGES
       "Hugepage allocator, as in GRAPE's Graphalytics runs; needs a preallocated hugetlb pool on the nodes (Linux)" OFF)

find_package(MPI REQUIRED COMPONENTS CXX)
set(THREADS_PREFER_PTHREAD_FLAG ON)
find_package(Threads REQUIRED)

# glog (GRAPE's logging), without gflags/unwind/tests
if(NOT TARGET glog::glog)
  set(WITH_GFLAGS OFF CACHE BOOL "" FORCE)
  set(WITH_GTEST OFF CACHE BOOL "" FORCE)
  set(WITH_UNWIND none CACHE STRING "" FORCE)
  set(BUILD_TESTING OFF CACHE BOOL "" FORCE)
  FetchContent_Declare(
    glog
    GIT_REPOSITORY https://github.com/google/glog.git
    GIT_TAG 7b134a5c82c0c0b5698bb6bf7a835b230c5638e4 # v0.7.1
    SYSTEM EXCLUDE_FROM_ALL)
  FetchContent_MakeAvailable(glog)
endif()

add_library(grape-lite STATIC grape/io/local_io_adaptor.cc grape/utils/memory_tracker.cc)
add_library(grape::grape-lite ALIAS grape-lite)
target_include_directories(grape-lite PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}
                                             ${CMAKE_CURRENT_SOURCE_DIR}/examples/analytical_apps)
target_include_directories(grape-lite SYSTEM PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/thirdparty)
target_compile_features(grape-lite PUBLIC cxx_std_14)
target_link_libraries(grape-lite PUBLIC MPI::MPI_CXX Threads::Threads glog::glog)
set_target_properties(grape-lite PROPERTIES POSITION_INDEPENDENT_CODE ON)
# upstream: -O3 in Release, -march=native outside macOS
target_compile_options(grape-lite INTERFACE $<$<NOT:$<CONFIG:Debug>>:-O3>
                                            $<$<AND:$<NOT:$<PLATFORM_ID:Darwin>>,$<NOT:$<CONFIG:Debug>>>:-march=native>)
if(GRAPE_USE_SIMD)
  target_compile_definitions(grape-lite PUBLIC USE_BMISS_STTNI_INTERSECT USE_SIMD_SORT)
  target_compile_options(grape-lite PUBLIC -mavx2)
endif()
if(GRAPE_USE_HUGEPAGES AND CMAKE_SYSTEM_NAME MATCHES "Linux")
  target_compile_definitions(grape-lite PUBLIC USE_HUGEPAGES)
endif()
