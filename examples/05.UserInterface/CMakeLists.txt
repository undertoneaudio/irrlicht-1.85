cmake_minimum_required(VERSION 3.16)

set(TARGET_NAME 05.UserInterface)

add_executable(${TARGET_NAME} main.cpp)

target_link_libraries(${TARGET_NAME} PRIVATE irrlicht)
target_include_directories(${TARGET_NAME} PRIVATE
    ${CMAKE_SOURCE_DIR}/include)
