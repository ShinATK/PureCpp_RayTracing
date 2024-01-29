cmake_minimum_required(VERSION 3.28)
project(RayTracing_In_OneWeekend)

# include
include_directories(./include)
# link_libraries("${PROJECT_SOURCE_DIR}/lib/libglfw3.a")#静态库文件
# aux_source_directory(./include/glad DIR_ALL_SRC)

set(CMAKE_CXX_STANDARD 11)

aux_source_directory(./src/ DIR_ALL_SRC)
add_executable(raytracing ${DIR_ALL_SRC})  
