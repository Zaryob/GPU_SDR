cmake_minimum_required(VERSION 3.10)

project(server LANGUAGES CXX CUDA)

# Set C++ and CUDA standards
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CUDA_STANDARD 17)

# Compiler flags
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -DBOOST_LOG_DYN_LINK -Wall -O3 -Ofast -march=native")
set(CMAKE_CUDA_FLAGS "${CMAKE_CUDA_FLAGS} -arch=sm_61")

set(THREADS_PREFER_PTHREAD_FLAG ON)

set(ENABLE_PYTHON_API ON)
set(ENABLE_EXAMPLES OFF)
set(ENABLE_UTILS OFF)
set(ENABLE_TESTS OFF)
# Include directories

# Linker libraries
add_subdirectory(uhd/host uhd)

#find_package(UHD REQUIRED)
find_package(CUDA REQUIRED)
find_package(CUDAToolkit REQUIRED)
find_package(Threads REQUIRED)
find_package(HDF5 REQUIRED COMPONENTS CXX HL )
find_package(Boost REQUIRED system program_options chrono thread log log_setup)
find_package(ZLIB)

# Source files
file(GLOB_RECURSE SRC_FILES cpp/*.cpp)
file(GLOB_RECURSE CUDA_SRC_FILES cpp/*.cu)

# Object files
add_library(obj_files STATIC ${SRC_FILES})
add_library(cuda_obj_files STATIC ${CUDA_SRC_FILES})

#message( FATAL_ERROR ${UHD_LIBRARIES})

include_directories(headers/
    ${Boost_INCLUDE_DIRS}
    ${CUDA_INCLUDE_DIRS}
    ${HDF5_INCLUDE_DIRS}
    "${UHD_SOURCE_DIR}/include"
    )


# Executable
add_executable(server $<TARGET_OBJECTS:obj_files> $<TARGET_OBJECTS:cuda_obj_files>)
set_property(TARGET server
             PROPERTY CUDA_SEPARABLE_COMPILATION ON)

# Link libraries
target_link_libraries(server PRIVATE
   ${Boost_LIBRARIES}
   uhd
   CUDA::cufft
   CUDA::cudart
   CUDA::cublas
   CUDA::culibos
   HDF5::HDF5
   Threads::Threads
   ZLIB::ZLIB)

# Documentation (Doxygen and Sphinx)
find_package(Doxygen)
if(DOXYGEN_FOUND)
    add_custom_target(doc_server_cpp
        COMMAND ${DOXYGEN_EXECUTABLE} ${CMAKE_SOURCE_DIR}/server_docs/doc_gen
        WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
        COMMENT "Generating C++ documentation with Doxygen..."
    )
endif()

find_package(Sphinx)
if(SPHINX_FOUND)
    add_custom_target(doc_server_python
        COMMAND ${SPHINX_EXECUTABLE} -b html ${CMAKE_SOURCE_DIR}/pyUSRP ${CMAKE_SOURCE_DIR}/lib_docs
        COMMENT "Generating Python documentation with Sphinx..."
    )
endif()

# Custom clean command
add_custom_target(clean_all
    COMMAND ${CMAKE_COMMAND} -P ${CMAKE_SOURCE_DIR}/cmake_clean.cmake
    COMMAND ${CMAKE_COMMAND} --build ${CMAKE_BINARY_DIR} --target clean
    COMMENT "Cleaning all generated files..."
)

# Clean script for additional cleaning tasks
file(WRITE ${CMAKE_SOURCE_DIR}/cmake_clean.cmake
"
file(REMOVE_RECURSE ${CMAKE_BINARY_DIR}/CMakeFiles)
file(REMOVE_RECURSE ${CMAKE_BINARY_DIR}/obj)
file(REMOVE ${CMAKE_BINARY_DIR}/server)
file(REMOVE ${CMAKE_BINARY_DIR}/Makefile)
file(REMOVE ${CMAKE_BINARY_DIR}/cmake_install.cmake)
file(REMOVE_RECURSE ${CMAKE_SOURCE_DIR}/lib_docs)
")

