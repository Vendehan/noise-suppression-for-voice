cmake_minimum_required(VERSION 3.6)
# Let project() manage VERSION variables (CMP0048)
if(POLICY CMP0048)
    cmake_policy(SET CMP0048 NEW)
endif()
project("Real-time Noise Suppression Plugin" VERSION 1.99 LANGUAGES C CXX)

include(GNUInstallDirs)

set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
set(CMAKE_BINARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

set(MINGW_ADDITIONAL_LINKING_FLAGS "-static-libgcc -static-libstdc++ -Wl,-Bstatic -lstdc++ -lpthread -Wl,-Bdynamic")

if(NOT BUILD_VERSION)
    set(BUILD_VERSION ${PROJECT_VERSION})
endif()

option(USE_SYSTEM_JUCE "" OFF)
option(BUILD_FOR_RELEASE "Additional optimizations and steps may be taken for release" OFF)
option(BUILD_TESTS "" ON)
option(ENABLE_NATIVE_OPTIMIZATIONS "Use -O3, -march=native and SIMD for Release builds (GCC/Clang)" ON)
option(BUILD_VST_PLUGIN "If the VST2 plugin should be built" ON)
option(BUILD_VST3_PLUGIN "If the VST3 plugin should be built" ON)
option(BUILD_LV2_PLUGIN "If the LV2 plugin should be built" ON)
option(BUILD_LADSPA_PLUGIN "If the LADSPA plugin should be built" ON)
option(BUILD_AU_PLUGIN "If the AU plugin should be built (macOS only)" ON)
option(BUILD_AUV3_PLUGIN "If the AUv3 plugin should be built (macOS only)" ON)
option(BUILD_RTCD "Enable x86 run-time CPU detection (x86 only)" OFF)

# Release build optimisations: -O3, -march=native, and optional SIMD (SSE2/AVX) when enabled
if(ENABLE_NATIVE_OPTIMIZATIONS AND (CMAKE_CXX_COMPILER_ID STREQUAL "GNU" OR CMAKE_CXX_COMPILER_ID MATCHES "Clang"))
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -O3 -march=native")
    set(CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE} -O3 -march=native")
    # SSE2/AVX are typically implied by -march=native on x86; explicit flags for clarity where needed
    if(CMAKE_SYSTEM_PROCESSOR MATCHES "x86|i686|AMD64|x86_64")
        set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -msse2")
    endif()
endif()

if (BUILD_TESTS)
    list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/external/catch2/)
endif ()

add_subdirectory(external/rnnoise)
add_subdirectory(src/common)
if (BUILD_LADSPA_PLUGIN)
    add_subdirectory(src/ladspa_plugin)
endif ()

if (BUILD_VST_PLUGIN OR BUILD_VST3_PLUGIN OR BUILD_LV2_PLUGIN OR BUILD_AU_PLUGIN OR BUILD_AUV3_PLUGIN)
    if (USE_SYSTEM_JUCE)
        find_package(JUCE)
    else ()
        if(NOT JUCE_SOURCE_DIR)
            set(JUCE_SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/external/JUCE CACHE PATH "Path to JUCE source (e.g. for JUCE 7.0.6+)")
        endif()
        # For install JUCE copies all its headers, no one needs them. It also doesn't install actual libraries.
        # On the other hand JUCE could install libraries during build process (see COPY_PLUGIN_AFTER_BUILD option).
        # So we have to manually install plugins.
        add_subdirectory(${JUCE_SOURCE_DIR} external/JUCE EXCLUDE_FROM_ALL)
    endif ()

    add_subdirectory(src/juce_plugin)
endif ()
