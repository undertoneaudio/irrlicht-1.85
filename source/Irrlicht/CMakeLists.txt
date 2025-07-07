cmake_minimum_required(VERSION 3.16)

# ─────────────── Irrlicht core library ─────────────────────────
add_library(irrlicht)

# 1️⃣ collect every source ----------------------------------------------------
file(GLOB_RECURSE IRR_SOURCES CONFIGURE_DEPENDS "*.cpp" "*.c" "*.mm")

# Exclude stuff the historic Makefile never built
list(FILTER IRR_SOURCES EXCLUDE REGEX ".*/examples/.*")
list(FILTER IRR_SOURCES EXCLUDE
     REGEX ".*/bzip2/(bzip2|bzip2recover|dlltest|unzcrash)\\.c$")
list(FILTER IRR_SOURCES EXCLUDE
     REGEX ".*/jpeglib/(cdjpeg|example|rdjpgcom|wrjpgcom|jpegtran|ansi2knr)\\.c$")
list(FILTER IRR_SOURCES EXCLUDE
     REGEX ".*/jpeglib/jmem(ansi|dos|mac|name)\\.c$")
list(FILTER IRR_SOURCES EXCLUDE REGEX ".*/zlib/gz.*\\.c$")  # drop gzip helpers

target_sources(irrlicht PRIVATE ${IRR_SOURCES})

# ───────────── include paths -------------------------------------------------
target_include_directories(irrlicht
    PUBLIC  $<BUILD_INTERFACE:${CMAKE_SOURCE_DIR}/include>
            $<INSTALL_INTERFACE:include/irrlicht>
    PRIVATE ${CMAKE_CURRENT_SOURCE_DIR}      # CIrrDeviceStub.h etc.
            ${CMAKE_CURRENT_SOURCE_DIR}/zlib
            ${CMAKE_CURRENT_SOURCE_DIR}/jpeglib
            ${CMAKE_CURRENT_SOURCE_DIR}/libpng)

# ───────────── compile flags & defines ---------------------------------------
target_compile_definitions(irrlicht
    PRIVATE IRRLICHT_EXPORTS=1 PNG_ARM_NEON_OPT=0)

if(APPLE)
    target_compile_options(irrlicht
        PRIVATE $<$<COMPILE_LANGUAGE:CXX>:-x objective-c++>)

    # Frameworks as one-piece arguments so CMake never turns them into -lXXX
    target_link_libraries(irrlicht PUBLIC
        "-framework Cocoa"
        "-framework Carbon"
        "-framework OpenGL"
        "-framework IOKit"
        "-framework ApplicationServices")

elseif(MSVC)
    target_compile_options(irrlicht PRIVATE /W4 /MP)

else()
    target_compile_options(irrlicht PRIVATE
        -Wall -pipe -fno-exceptions -fno-rtti -fstrict-aliasing)
    find_package(OpenGL REQUIRED)
    find_package(X11    REQUIRED COMPONENTS X11 Xxf86vm)
    target_link_libraries(irrlicht PUBLIC OpenGL::GL X11::X11 X11::Xxf86vm)
endif()

if(WIN32)
    target_link_libraries(irrlicht PUBLIC gdi32 opengl32 winmm)
endif()

# ───────────── output name & versioning --------------------------------------
set_target_properties(irrlicht PROPERTIES
    OUTPUT_NAME Irrlicht
    VERSION     ${PROJECT_VERSION}
    SOVERSION   ${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR})

# ───────────── install rules --------------------------------------------------
install(TARGETS irrlicht
        ARCHIVE DESTINATION lib
        LIBRARY DESTINATION lib
        RUNTIME DESTINATION bin)

install(DIRECTORY ${CMAKE_SOURCE_DIR}/include/
        DESTINATION include/irrlicht)
