cmake_minimum_required(VERSION 3.18)
project(sbachess C)

add_library(sbachess SHARED src/mod.c)
target_include_directories(sbachess PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}/../..)
target_link_libraries(sbachess log)
