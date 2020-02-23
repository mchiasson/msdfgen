cmake_minimum_required(VERSION 3.2)

list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake)

include(HunterGate)
HunterGate(
    URL "https://github.com/ruslo/hunter/archive/v0.23.30.tar.gz"
    SHA1 "c062b0a2316cf8dcb57b6a9ad49fea1025a928f6"
)

project(msdfgen VERSION 1.5)

hunter_add_package(freetype)
hunter_add_package(tinyxml2)

find_package(freetype CONFIG REQUIRED)
find_package(tinyxml2 CONFIG REQUIRED)

# Make release mode default (turn on optimizations)
if(NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release)
endif()

#----------------------------------------------------------------
# Support Functions
#----------------------------------------------------------------

# Mirror the folder structure for sources inside the IDE...
function(folderize_sources sources prefix)
	foreach(FILE ${${sources}}) 
	  get_filename_component(PARENT_DIR "${FILE}" PATH)

	  # skip src or include and changes /'s to \\'s
	  string(REPLACE "${prefix}" "" GROUP "${PARENT_DIR}")
	  string(REPLACE "/" "\\" GROUP "${GROUP}")

      # If it's got a path, then append a "\\" separator (otherwise leave it blank)
	  if ("${GROUP}" MATCHES ".+")
	  	set(GROUP "\\${GROUP}")
	  endif()

	  source_group("${GROUP}" FILES "${FILE}")
	endforeach()
endfunction(folderize_sources)



file(GLOB_RECURSE msdfgen_HEADERS
	"core/*.h"
	"core/*.hpp"
	"lib/*.h"
	"ext/*.h"
	"include/*.h"
)

file(GLOB_RECURSE msdfgen_SOURCES
	"core/*.cpp"
	"lib/*.cpp"
	"ext/*.cpp"
)

# Build the library (aliased name because it's the same target name the exe)
folderize_sources(msdfgen_HEADERS ${CMAKE_SOURCE_DIR})
folderize_sources(msdfgen_SOURCES ${CMAKE_SOURCE_DIR})

add_library(lib_msdfgen ${msdfgen_SOURCES} ${msdfgen_HEADERS})
set_target_properties(lib_msdfgen PROPERTIES
    OUTPUT_NAME msdfgen
    CXX_STANDARD 11
    CXX_STANDARD_REQUIRED YES
)

target_link_libraries(lib_msdfgen PUBLIC freetype::freetype tinyxml2)
target_include_directories(lib_msdfgen PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include> $<INSTALL_INTERFACE:include>)
target_compile_definitions(lib_msdfgen PUBLIC MSDFGEN_USE_CPP11=1)

find_package(OpenMP)
if (TARGET OpenMP::OpenMP_CXX)
    target_link_libraries(lib_msdfgen PUBLIC OpenMP::OpenMP_CXX)
    target_compile_definitions(lib_msdfgen PUBLIC MSDFGEN_USE_OPENMP=1)
endif()


# Build the executable

add_executable(msdfgen main.cpp msdfgen.h msdfgen-ext.h)
target_compile_definitions(msdfgen PRIVATE MSDFGEN_STANDALONE)
target_link_libraries(msdfgen lib_msdfgen)

#----------------------------------------------------------------
# Install
#----------------------------------------------------------------

# Install public headers
install(
    FILES
        msdfgen.h
        msdfgen-ext.h
    DESTINATION
        include/msdfgen/
)
install(
    FILES
        core/arithmetics.hpp
        core/Bitmap.h
        core/Bitmap.hpp
        core/BitmapRef.hpp
        core/contour-combiners.h
        core/Contour.h
        core/EdgeColor.h
        core/edge-coloring.h
        core/EdgeHolder.h
        core/edge-segments.h
        core/edge-selectors.h
        core/equation-solver.h
        core/estimate-sdf-error.h
        core/pixel-conversion.hpp
        core/rasterization.h
        core/render-sdf.h
        core/save-bmp.h
        core/save-tiff.h
        core/Scanline.h
        core/shape-description.h
        core/Shape.h
        core/SignedDistance.h
        core/Vector2.h
    DESTINATION
        include/msdfgen/core/
)
install(
    FILES
        ext/import-font.h
	ext/import-svg.h
	ext/save-png.h
    DESTINATION
        include/msdfgen/ext/
)
install(
    FILES
        include/lodepng.h
    DESTINATION
        include/
)

# Create the CMake version file.
include(CMakePackageConfigHelpers)
write_basic_package_version_file(
  "${CMAKE_CURRENT_BINARY_DIR}/msdfgenConfigVersion.cmake"
  VERSION ${PACKAGE_VERSION}
  COMPATIBILITY AnyNewerVersion
)

set(targets_export_name "msdfgenTargets")

# Create the Config file.
include(CMakePackageConfigHelpers)
set(ConfigPackageLocation lib/cmake/msdfgen)
configure_package_config_file(
  ${CMAKE_CURRENT_SOURCE_DIR}/cmake/msdfgenConfig.cmake.in
  ${CMAKE_CURRENT_BINARY_DIR}/msdfgenConfig.cmake
  INSTALL_DESTINATION ${ConfigPackageLocation}
)

install(TARGETS lib_msdfgen
    EXPORT ${targets_export_name}
    LIBRARY DESTINATION lib COMPONENT Runtime
    ARCHIVE DESTINATION lib COMPONENT Development
    RUNTIME DESTINATION bin COMPONENT Runtime
    BUNDLE DESTINATION bin COMPONENT Runtime
    PUBLIC_HEADER DESTINATION include COMPONENT Development
    BUNDLE DESTINATION bin COMPONENT Runtime
)

# Install the generated CMake files.
install(EXPORT ${targets_export_name} NAMESPACE "msdfgen::" DESTINATION lib/cmake/msdfgen)
install(
  FILES "${CMAKE_CURRENT_BINARY_DIR}/msdfgenConfigVersion.cmake"
        "${CMAKE_CURRENT_BINARY_DIR}/msdfgenConfig.cmake"
  DESTINATION ${ConfigPackageLocation}
)
