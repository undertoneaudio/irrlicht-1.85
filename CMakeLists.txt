cmake_minimum_required(VERSION 3.16)

# ──────────────────────────────────────────────────────────────
#  Irrlicht Engine (C++ / Objective-C++) – CMake build
# ──────────────────────────────────────────────────────────────
project(Irrlicht
        VERSION 1.8.5
        LANGUAGES C CXX OBJCXX)

# Options
option(BUILD_SHARED_LIBS     "Build Irrlicht as a shared library" OFF)
option(IRR_BUILD_EXAMPLES    "Build examples"                     ON)

# Library
add_subdirectory(source/Irrlicht)

# Examples
if(IRR_BUILD_EXAMPLES)
    add_subdirectory(examples/05.UserInterface)
endif()
