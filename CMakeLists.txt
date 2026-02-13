cmake_minimum_required(VERSION 3.31 FATAL_ERROR)

# Ensure CUDA architectures must be set (CMP0104)
cmake_policy(SET CMP0104 NEW)

project(leapct LANGUAGES CXX VERSION 1.26)

# =============================================================================
# Build Options
# =============================================================================
option(LEAPCT_USE_CPU "Force CPU-only build (disable CUDA even if available)" OFF)
option(LEAPCT_USE_GPU "Force GPU build (fail if CUDA not found)" OFF)

# =============================================================================
# GPU/CUDA Detection and Configuration
# =============================================================================
set(LEAPCT_ENABLE_CUDA OFF)

if(LEAPCT_USE_CPU AND LEAPCT_USE_GPU)
  message(FATAL_ERROR "Cannot specify both LEAPCT_USE_CPU and LEAPCT_USE_GPU")
endif()

if(LEAPCT_USE_CPU)
  message(STATUS "LEAPCT: Forcing CPU-only build (LEAPCT_USE_CPU=ON)")
elseif(LEAPCT_USE_GPU)
  # User explicitly requested GPU - fail if not found
  find_package(CUDAToolkit REQUIRED)
  enable_language(CUDA)
  set(LEAPCT_ENABLE_CUDA ON)
  message(STATUS "LEAPCT: GPU build enabled (user requested)")
else()
  # Auto-detect CUDA
  find_package(CUDAToolkit QUIET)
  if(CUDAToolkit_FOUND)
    enable_language(CUDA)
    set(LEAPCT_ENABLE_CUDA ON)
    message(STATUS "LEAPCT: CUDA ${CUDAToolkit_VERSION} detected - enabling GPU build")
  else()
    message(STATUS "LEAPCT: CUDA toolkit not found - using CPU-only build")
  endif()
endif()

# =============================================================================
# Compiler Settings
# =============================================================================
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_VISIBILITY_PRESET "hidden")
set(CMAKE_VISIBILITY_INLINES_HIDDEN TRUE)

if(LEAPCT_ENABLE_CUDA)
  set(CMAKE_CUDA_STANDARD 17)
  set(CMAKE_CUDA_STANDARD_REQUIRED ON)

  # CMake 3.24+: Use 'native' to auto-detect GPU architecture at build time
  # This replaces hardcoded values like "52;60;70;75;80;86"
  if(NOT DEFINED CMAKE_CUDA_ARCHITECTURES)
    set(CMAKE_CUDA_ARCHITECTURES native)
  endif()
endif()

# =============================================================================
# Dependencies
# =============================================================================
include(GenerateExportHeader)
enable_testing()

find_package(OpenMP REQUIRED)

if(WIN32)
  set(CMAKE_SHARED_LIBRARY_PREFIX "lib")
  set(CMAKE_STATIC_LIBRARY_PREFIX "lib")
  add_compile_definitions("WIN32")
endif()

# =============================================================================
# Source Files
# =============================================================================

set(LEAPCT_COMMON_HEADERS
  src/tomographic_models.h
  src/list_of_tomographic_models.h
  src/tomographic_models_c_interface.h
  src/parameters.h
  src/projectors.h
  src/filtered_backprojection.h
  src/projectors_SF_cpu.h
  src/projectors_Joseph_cpu.h
  src/projectors_symmetric_cpu.h
  src/projectors_Siddon_cpu.h
  src/sensitivity_cpu.h
  src/ramp_filter_cpu.h
  src/ray_weighting_cpu.h
  src/find_center_cpu.h
  src/sinogram_replacement.h
  src/resample_cpu.h
  src/cpu_utils.h
  src/phantom.h
  src/analytic_ray_tracing.h
  src/rebin.h
  src/file_io.h
  src/leap_defines.h
  src/log.h
)

set(LEAPCT_GPU_HEADERS
  src/projectors_SF.cuh
  src/projectors_extendedSF.cuh
  src/projectors_Joseph.cuh
  src/projectors_symmetric.cuh
  src/projectors_attenuated.cuh
  src/projectors_Siddon.cuh
  src/backprojectors_VD.cuh
  src/sensitivity.cuh
  src/ramp_filter.cuh
  src/ray_weighting.cuh
  src/scatter_models.cuh
  src/noise_filters.cuh
  src/total_variation.cuh
  src/matching_pursuit.cuh
  src/bilateral_filter.cuh
  src/guided_filter.cuh
  src/geometric_calibration.cuh
  src/resample.cuh
  src/cuda_utils.h
  src/analytic_ray_tracing_gpu.cuh
)

set(LEAPCT_COMMON_SOURCES
  src/tomographic_models.cpp
  src/list_of_tomographic_models.cpp
  src/tomographic_models_c_interface.cpp
  src/parameters.cpp
  src/projectors.cpp
  src/filtered_backprojection.cpp
  src/projectors_SF_cpu.cpp
  src/projectors_Joseph_cpu.cpp
  src/projectors_symmetric_cpu.cpp
  src/projectors_Siddon_cpu.cpp
  src/sensitivity_cpu.cpp
  src/ramp_filter_cpu.cpp
  src/ray_weighting_cpu.cpp
  src/find_center_cpu.cpp
  src/sinogram_replacement.cpp
  src/resample_cpu.cpp
  src/cpu_utils.cpp
  src/phantom.cpp
  src/analytic_ray_tracing.cpp
  src/rebin.cpp
  src/file_io.cpp
)

set(LEAPCT_GPU_SOURCES
  src/projectors_SF.cu
  src/projectors_extendedSF.cu
  src/projectors_Joseph.cu
  src/projectors_symmetric.cu
  src/projectors_attenuated.cu
  src/projectors_Siddon.cu
  src/backprojectors_VD.cu
  src/sensitivity.cu
  src/ramp_filter.cu
  src/resample.cu
  src/noise_filters.cu
  src/total_variation.cu
  src/matching_pursuit.cu
  src/bilateral_filter.cu
  src/guided_filter.cu
  src/geometric_calibration.cu
  src/ray_weighting.cu
  src/scatter_models.cu
  src/analytic_ray_tracing_gpu.cu
  src/cuda_utils.cu
)

# =============================================================================
# Library Target
# =============================================================================
add_library(leapct SHARED)
generate_export_header(leapct)

target_sources(leapct
  PRIVATE
    ${LEAPCT_COMMON_SOURCES}
  PUBLIC
    FILE_SET common_headers
    TYPE HEADERS
    BASE_DIRS src
    FILES ${LEAPCT_COMMON_HEADERS}
)

if(LEAPCT_ENABLE_CUDA)
  target_sources(leapct
    PRIVATE
      ${LEAPCT_GPU_SOURCES}
    PUBLIC
      FILE_SET gpu_headers
      TYPE HEADERS
      BASE_DIRS src
      FILES ${LEAPCT_GPU_HEADERS}
  )
  target_link_libraries(leapct PRIVATE CUDA::cudart)
else()
  target_compile_definitions(leapct PRIVATE __USE_CPU)
endif()

target_link_libraries(leapct PRIVATE OpenMP::OpenMP_CXX)
target_include_directories(leapct PUBLIC 
    "$<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>"
)

# =============================================================================
# Installation
# =============================================================================
include(GNUInstallDirs)

install(TARGETS leapct
  FILE_SET common_headers
  RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
  ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
  LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
)

if(LEAPCT_ENABLE_CUDA)
  install(TARGETS leapct FILE_SET gpu_headers)
endif()

find_package(Python3 COMPONENTS Interpreter REQUIRED)
install(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/src/
    DESTINATION ${Python3_SITELIB}
    FILES_MATCHING PATTERN "*.py"
    PATTERN "__pycache__" EXCLUDE
)
# =============================================================================
# Summary
# =============================================================================
message(STATUS "")
message(STATUS "LEAPCT Configuration Summary:")
message(STATUS "  CMake version:      ${CMAKE_VERSION}")
message(STATUS "  Build type:         ${CMAKE_BUILD_TYPE}")
message(STATUS "  CUDA enabled:       ${LEAPCT_ENABLE_CUDA}")
if(LEAPCT_ENABLE_CUDA)
  message(STATUS "  CUDA version:       ${CUDAToolkit_VERSION}")
  message(STATUS "  CUDA compiler:      ${CMAKE_CUDA_COMPILER}")
  message(STATUS "  CUDA architectures: ${CMAKE_CUDA_ARCHITECTURES}")
endif()
message(STATUS "")
