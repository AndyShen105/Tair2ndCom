cmake_minimum_required(VERSION 3.16)
project(tair)

set(CMAKE_CXX_STANDARD 14)

include_directories(include)
include_directories(judge)
include_directories(nvm_engine)
include_directories(nvm_example)
include_directories(yet_another_nvm_example)

add_executable(tair
        nvm_engine/concurrentqueue.h
        nvm_engine/NvmEngine.cpp
        nvm_engine/NvmEngine.hpp
        test/test.cpp)
