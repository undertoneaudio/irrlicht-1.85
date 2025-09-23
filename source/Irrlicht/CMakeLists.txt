cmake_minimum_required(VERSION 3.16)

# ─────────────── Irrlicht core library ─────────────────────────
add_library(irrlicht)

# Use a pre-C++17 GNU dialect for Irrlicht only (avoids __STRICT_ANSI__ and register errors)
set_target_properties(irrlicht PROPERTIES
  CXX_STANDARD 14
  CXX_STANDARD_REQUIRED OFF
  CXX_EXTENSIONS ON)           # gnu++14 instead of c++14

# Make sure ObjC++ files (.mm) also get the same -std (some CMake gens won’t apply CXX_STANDARD to OBJCXX)
target_compile_options(irrlicht PRIVATE
  $<$<OR:$<COMPILE_LANGUAGE:CXX>,$<COMPILE_LANGUAGE:OBJCXX>>:-std=gnu++14>)

# Safety net: if some toolchain still defines __STRICT_ANSI__, just undefine it for Irrlicht
if(CMAKE_CXX_COMPILER_ID MATCHES "Clang|AppleClang|GNU")
  target_compile_options(irrlicht PRIVATE -U__STRICT_ANSI__)
endif()

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

# platform specific cleanup
if(APPLE)
    list(FILTER IRR_SOURCES EXCLUDE REGEX ".*/CD3D.*\\.cpp$")          # D3D9/11 stuff
    list(FILTER IRR_SOURCES EXCLUDE REGEX ".*/CIrrDeviceWin32\\.cpp$") # Win32 device
elseif(WIN32)
    list(FILTER IRR_SOURCES EXCLUDE REGEX ".*/CIrrDeviceMacOSX\\.mm$")
endif()

target_sources(irrlicht PRIVATE ${IRR_SOURCES})

# --- Suppress selected warnings for Irrlicht (only) -------------------------
include(CheckCCompilerFlag)
include(CheckCXXCompilerFlag)

# Helper: add a flag if the current compiler supports it
function(_irr_add_flag_if_supported tgt lang flag)
  # Make a unique, cache-friendly var name for the probe
  string(MAKE_C_IDENTIFIER "${tgt}_${lang}_${flag}" _var)
  if(${lang} STREQUAL C)
    check_c_compiler_flag("${flag}" ${_var})
    if(${_var})
      target_compile_options(${tgt} PRIVATE $<$<COMPILE_LANGUAGE:C>:${flag}>)
    endif()
  else() # treat CXX and OBJCXX together (same compiler front-end on Apple)
    check_cxx_compiler_flag("${flag}" ${_var})
    if(${_var})
      target_compile_options(${tgt} PRIVATE
        $<$<OR:$<COMPILE_LANGUAGE:CXX>,$<COMPILE_LANGUAGE:OBJCXX>>:${flag}>)
    endif()
  endif()
endfunction()

# The warnings you asked to silence
set(_irr_flags_c
  -Wno-unused-parameter
  -Wno-unused-function
  -Wno-deprecated-declarations
)

set(_irr_flags_cxx
  -Wno-unused-parameter
  -Wno-unused-function
  -Wno-unused-private-field
  -Wno-deprecated-declarations
  -Wno-deprecated-copy
  -Wno-deprecated-copy-with-user-provided-copy
)

foreach(f IN LISTS _irr_flags_c)
  _irr_add_flag_if_supported(irrlicht C   "${f}")
endforeach()

foreach(f IN LISTS _irr_flags_cxx)
  _irr_add_flag_if_supported(irrlicht CXX "${f}")   # applies to CXX & OBJCXX
endforeach()

# macOS OpenGL deprecation messages (headers)
if(APPLE)
  target_compile_definitions(irrlicht PRIVATE GL_SILENCE_DEPRECATION=1)
endif()

# Optional: MSVC equivalents (no probing needed)
if(MSVC)
  target_compile_options(irrlicht PRIVATE
    /wd4100   # unreferenced formal parameter
    /wd4505   # unreferenced (local) function removed
    /wd4996   # deprecated declarations
  )
  # MSVC doesn't have direct equivalents for -Wunused-private-field or
  # -Wdeprecated-copy(-with-user-provided-copy); there isn't a precise switch.
endif()

# ───────────── include paths -------------------------------------------------
target_include_directories(irrlicht SYSTEM
    PUBLIC  $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>
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
elseif(WIN32)
    target_link_libraries(irrlicht PUBLIC gdi32 opengl32 winmm)
else()
    target_compile_options(irrlicht PRIVATE
        -Wall -pipe -fno-exceptions -fno-rtti -fstrict-aliasing)

    find_package(PkgConfig REQUIRED)

    pkg_check_modules(GL_PC REQUIRED gl)
    pkg_check_modules(X11_PC REQUIRED x11)
    pkg_check_modules(XXF86VM_PC REQUIRED xxf86vm)

    target_include_directories(irrlicht PUBLIC
        ${X11_PC_INCLUDE_DIRS}
        ${XXF86VM_PC_INCLUDE_DIRS}
        ${GL_PC_INCLUDE_DIRS}
    )

    target_link_libraries(irrlicht PUBLIC
        ${X11_PC_LINK_LIBRARIES}
        ${XXF86VM_PC_LINK_LIBRARIES}
        ${GL_PC_LINK_LIBRARIES}
        )
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
