# (c) 2008 by BogDan Vatra < bogdan@licentia.eu >
# vim: set ts=4 sw=4 et :

cmake_minimum_required(VERSION 3.0)
project(zint-package)

set(CMAKE_INCLUDE_CURRENT_DIR ON)

#set(CMAKE_VERBOSE_MAKEFILE ON)
#comment or remove the above line before release

set (ZINT_VERSION_MAJOR 2)
set (ZINT_VERSION_MINOR 9)
set (ZINT_VERSION_RELEASE 1)
set (ZINT_VERSION_BUILD 9) # Set to 0 before release, set to 9 after release
set (ZINT_VERSION "${ZINT_VERSION_MAJOR}.${ZINT_VERSION_MINOR}.${ZINT_VERSION_RELEASE}.${ZINT_VERSION_BUILD}" )

add_definitions(-DZINT_VERSION=\"${ZINT_VERSION}\")

set(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake/modules" )

set(ZINT_DEBUG FALSE CACHE BOOL "Set debug compile flag")
set(ZINT_SANITIZE FALSE CACHE BOOL "Set sanitize compile/link flags")
set(ZINT_TEST FALSE CACHE BOOL "Set test compile flag")
set(ZINT_STATIC FALSE CACHE BOOL "Build static library")

include (SetPaths.cmake)

include(CheckCXXCompilerFlag)

check_cxx_compiler_flag("-Wall" CXX_COMPILER_FLAG_WALL)
if(CXX_COMPILER_FLAG_WALL)
    add_compile_options("-Wall")
endif()

check_cxx_compiler_flag("-Wextra" CXX_COMPILER_FLAG_WEXTRA)
if(CXX_COMPILER_FLAG_WEXTRA)
    add_compile_options("-Wextra")
endif()

check_cxx_compiler_flag("-Wpedantic" CXX_COMPILER_FLAG_WPEDANTIC)
if(CXX_COMPILER_FLAG_WPEDANTIC)
    add_compile_options("-Wpedantic")
endif()

if(ZINT_DEBUG)
    check_cxx_compiler_flag("-g" CXX_COMPILER_FLAG_G)
    if(CXX_COMPILER_FLAG_G)
        add_compile_options("-g")
    endif()
    check_cxx_compiler_flag("-O0" CXX_COMPILER_FLAG_O0)
    if(CXX_COMPILER_FLAG_O0)
        add_compile_options("-O0")
    endif()
endif()

if(CMAKE_COMPILER_IS_GNUCXX OR CMAKE_COMPILER_IS_GNUCC)
    if(ZINT_SANITIZE)
        add_compile_options(-fsanitize=undefined -fsanitize=address)
        link_libraries(-fsanitize=undefined -fsanitize=address)
        # Gives warning on MainWindow::setupUI() and retries (and takes forever) if var-tracking-assignments enabled
        add_compile_options(-fno-var-tracking-assignments)
    endif()
endif()

if(ZINT_TEST)
    add_definitions("-DZINT_TEST")
endif()

IF(APPLE)
    IF (UNIVERSAL) # TODO: make universal binary
        IF(NOT ZINT_HAS_BEEN_RUN_BEFORE)
            IF(EXISTS /Developer/SDKs/MacOSX10.5.sdk OR EXISTS /SDKs/MacOSX10.5.sdk)
                SET(CMAKE_OSX_ARCHITECTURES "ppc;i386;ppc64;x86_64" CACHE STRING "Build architectures for OSX" FORCE)
                SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fvisibility-inlines-hidden -Wl -single_module " CACHE STRING "Flags used by the compiler during all build types." FORCE)
            ELSE(EXISTS /Developer/SDKs/MacOSX10.5.sdk OR EXISTS /SDKs/MacOSX10.5.sdk)
                IF(EXISTS /Developer/SDKs/MacOSX10.4u.sdk OR EXISTS /SDKs/MacOSX10.4u.sdk)
                    SET(CMAKE_OSX_ARCHITECTURES "ppc;i386" CACHE STRING "Build architectures for OSX" FORCE)
                    SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fvisibility-inlines-hidden -Wl -single_module " CACHE STRING "Flags used by the compiler during all build types." FORCE)
               ENDIF(EXISTS /Developer/SDKs/MacOSX10.4u.sdk OR EXISTS /SDKs/MacOSX10.4u.sdk)
            ENDIF(EXISTS /Developer/SDKs/MacOSX10.5.sdk OR EXISTS /SDKs/MacOSX10.5.sdk)
            message("Build architectures for OSX:${CMAKE_OSX_ARCHITECTURES}")
        ENDIF(NOT ZINT_HAS_BEEN_RUN_BEFORE)

    ELSE (UNIVERSAL)
        SET(CMAKE_OSX_SYSROOT "/")
    ENDIF (UNIVERSAL)

    IF (PNG_FOUND)
         INCLUDE_DIRECTORIES(
            "${PNG_INCLUDE_DIR}"
         )
        SET(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -lpng")
        SET(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_SHARED_LINKER_FLAGS} -lpng")
        SET(CMAKE_MODULE_LINKER_FLAGS "${CMAKE_MODULE_LINKER_FLAGS} -lpng")
    ENDIF (PNG_FOUND)

ENDIF(APPLE)

add_subdirectory(backend)
add_subdirectory(frontend)

if($ENV{CMAKE_PREFIX_PATH} MATCHES "6[.][0-9][.][0-9]")
    set(USE_QT6 TRUE)
    message(STATUS "Using Qt6")
    cmake_policy(SET CMP0012 NEW) # Recognize constants in if()
    cmake_policy(SET CMP0072 NEW) # Choose OpenGL over legacy GL
    find_package(Qt6Widgets)
    find_package(Qt6Gui)
    find_package(Qt6UiTools)

    if(Qt6Widgets_FOUND AND Qt6Gui_FOUND AND Qt6UiTools_FOUND)
        set(QT_USE_QTGUI TRUE)
        set(QT_USE_QTUITOOLS TRUE)
        set(QT_USE_QTXML TRUE)
        include_directories(
            ${CMAKE_CURRENT_SOURCE_DIR}
            ${Qt6Widgets_INCLUDES}
            ${Qt6Gui_INCLUDES}
            ${Qt6UiTools_INCLUDES}
            ${CMAKE_CURRENT_BINARY_DIR}
            )
        add_subdirectory(frontend_qt)
    else()
        message(STATUS "Could NOT find Qt6")
    endif()
else()
    message(STATUS "Using Qt5")
    find_package(Qt5Widgets)
    find_package(Qt5Gui)
    find_package(Qt5UiTools)

    if(Qt5Widgets_FOUND AND Qt5Gui_FOUND AND Qt5UiTools_FOUND)
        set(QT_USE_QTGUI TRUE)
        set(QT_USE_QTUITOOLS TRUE)
        set(QT_USE_QTXML TRUE)
        include_directories(
            ${CMAKE_CURRENT_SOURCE_DIR}
            ${Qt5Widgets_INCLUDES}
            ${Qt5Gui_INCLUDES}
            ${Qt5UiTools_INCLUDES}
            ${CMAKE_CURRENT_BINARY_DIR}
            )
        add_subdirectory(frontend_qt)
    else()
        message(STATUS "Could NOT find Qt5")
    endif()
endif()

configure_file(
  "${CMAKE_CURRENT_SOURCE_DIR}/cmake_uninstall.cmake.in"
  "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake"
  IMMEDIATE @ONLY)

add_custom_target(uninstall
  "${CMAKE_COMMAND}" -P "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake")

# staniek: don't install 
if (DATA_INSTALL_DIR)
    set(CMAKE_MODULES_INSTALL_PATH ${DATA_INSTALL_DIR}/cmake/modules)
else (DATA_INSTALL_DIR)
    set(CMAKE_MODULES_INSTALL_PATH ${CMAKE_ROOT}/Modules)
endif(DATA_INSTALL_DIR)

install(FILES cmake/modules/FindZint.cmake DESTINATION ${CMAKE_MODULES_INSTALL_PATH} COMPONENT Devel)

# This needs to be run very last so other parts of the scripts can take
# advantage of this.
IF(NOT ZINT_HAS_BEEN_RUN_BEFORE)
    SET(ZINT_HAS_BEEN_RUN_BEFORE 1 CACHE INTERNAL "Flag to track whether this is the first time running CMake or if CMake has been configured before")
ENDIF(NOT ZINT_HAS_BEEN_RUN_BEFORE)
