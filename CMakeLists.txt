cmake_minimum_required(VERSION 3.0)
project(mymuduo)

# cmake => makefile   make
# mymuduo最终编译成so动态库，设置动态库的路径，放在根目录的lib文件夹下面
set(LIBRARY_OUTPUT_PATH ${PROJECT_SOURCE_DIR}/lib)
# 设置调试信息 以及 启动C++11语言标准
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -g -std=c++11 -fPIC")

# 配置头文件搜索路径
include_directories(${PROJECT_SOURCE_DIR}/include)

# 配置待编译的源文件路径
aux_source_directory(${PROJECT_SOURCE_DIR}/src SRC)

# 编译生成动态库mymuduo
add_library(mymuduo SHARED ${SRC})