cmake_minimum_required(VERSION 3.2)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED TRUE)
list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake)

project(bx VERSION 1.0.0)

if(CMAKE_BUILD_TYPE MATCHES "Debug" OR CMAKE_BUILD_TYPE MATCHES "RelWithDebInfo")
    option(BX_AMALGAMATED "Use amalgamated.cpp" FALSE)
else()
    option(BX_AMALGAMATED "Use amalgamated.cpp" TRUE)
endif()

# Grab the bx source files
file(GLOB BX_SOURCES src/*.cpp)

if(BX_AMALGAMATED)
    set(BX_NOBUILD ${BX_SOURCES})
    list(REMOVE_ITEM BX_NOBUILD ${PROJECT_SOURCE_DIR}/src/amalgamated.cpp)
    foreach(BX_SRC ${BX_NOBUILD})
        set_source_files_properties(${BX_SRC} PROPERTIES HEADER_FILE_ONLY ON)
    endforeach()
else()
    set_source_files_properties(${PROJECT_SOURCE_DIR}/src/amalgamated.cpp PROPERTIES HEADER_FILE_ONLY ON)
endif()

add_library(bx ${BX_SOURCES})

if(MSVC)
    target_include_directories(bx
        PUBLIC
            $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>
            $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include/compat/msvc>
            $<INSTALL_INTERFACE:include>
            $<INSTALL_INTERFACE:include/compat/msvc>
        PRIVATE 3rdparty
    )
elseif(MINGW)
    target_include_directories(bx
        PUBLIC
            $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>
            $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include/compat/mingw>
            $<INSTALL_INTERFACE:include>
            $<INSTALL_INTERFACE:include/compat/mingw>
        PRIVATE 3rdparty
    )
elseif(APPLE)
    target_include_directories(bx
        PUBLIC
            $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>
            $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include/compat/osx>
            $<INSTALL_INTERFACE:include>
            $<INSTALL_INTERFACE:include/compat/osx>
        PRIVATE 3rdparty
    )
else()
    target_include_directories(bx
        PUBLIC
            $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>
            $<INSTALL_INTERFACE:include>
        PRIVATE 3rdparty
    )
endif()

target_compile_definitions(bx
    PUBLIC
        __STDC_LIMIT_MACROS
        __STDC_FORMAT_MACROS
        __STDC_CONSTANT_MACROS
        $<$<CONFIG:Debug>:BX_CONFIG_DEBUG=1>
)

if(MSVC OR MINGW)
    target_link_libraries(bx PUBLIC psapi)
elseif(APPLE)
    find_library(FOUNDATION_LIBRARY Foundation)
    target_link_libraries(bx PUBLIC ${FOUNDATION_LIBRARY})
elseif(UNIX)
    find_package(Threads)
    target_link_libraries(bx PUBLIC ${CMAKE_THREAD_LIBS_INIT} dl rt)
endif()

set_target_properties( bx PROPERTIES FOLDER "bkaradzic" )

set(targets_export_name "bxTargets")
set(ConfigPackageLocation lib/cmake/bx)

# Create the CMake version file.
include(CMakePackageConfigHelpers)
write_basic_package_version_file(
    ${CMAKE_CURRENT_BINARY_DIR}/bxConfigVersion.cmake
    VERSION ${PACKAGE_VERSION}
    COMPATIBILITY AnyNewerVersion
)

# Create the Config file.
include(CMakePackageConfigHelpers)
configure_package_config_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/cmake/bxConfig.cmake.in
    ${CMAKE_CURRENT_BINARY_DIR}/bxConfig.cmake
    INSTALL_DESTINATION ${ConfigPackageLocation}
)

# Install the different headers and libraries.
install(
    DIRECTORY
        include/bx
        include/compat
        include/tinystl
    DESTINATION include COMPONENT Development
)

install(TARGETS bx
    EXPORT ${targets_export_name}
    LIBRARY DESTINATION lib COMPONENT Runtime
    ARCHIVE DESTINATION lib COMPONENT Development
    RUNTIME DESTINATION bin COMPONENT Runtime
    BUNDLE DESTINATION bin COMPONENT Runtime
    PUBLIC_HEADER DESTINATION include COMPONENT Development
    BUNDLE DESTINATION bin COMPONENT Runtime
)

# Install the generated CMake files.
install(
    EXPORT
        ${targets_export_name}
    NAMESPACE
        "bkaradzic::"
    DESTINATION
        ${ConfigPackageLocation}
)

install(
    FILES
        ${CMAKE_CURRENT_BINARY_DIR}/bxConfigVersion.cmake
        ${CMAKE_CURRENT_BINARY_DIR}/bxConfig.cmake
    DESTINATION
        ${ConfigPackageLocation}
)

