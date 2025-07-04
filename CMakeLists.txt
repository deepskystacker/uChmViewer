# uChmViewer build script with cmake
#
#       Common option:
# USE_KF5            - if defined, build with KDE Framework 5.
# USE_STATIC_CHMLIB  - if defined, static linkage to the chm library is used.
# USE_WEBENGINE      - if defined, WebEngine is used, otherwise WebKit is used.
# USE_GETTEXT        - if defined, then GNU Gettext will be used for translation.
#                      The default is ON.
# USE_DBUS           - if defined, build with D-Bus integration.
#                      The default is ON for Linux.
# USE_MAC_APP        - if defined, build with  derived QApplication.
#                      The default is ON for macOs.
#
#       System-specific option:
# USE_DEPLOY_RUNTIME - if defined, the runtime dependencies will be copied to the
#                      folder with the executable. This option is applicable for
#                      Windows and macOS.
# USE_MACOS_BUNDLE   - if defined, configure installation for macOS as a bundle.
#                      The default is ON.
################################################

cmake_minimum_required(VERSION 3.13)

# Project info
project(uchmviewer HOMEPAGE_URL "https://github.com/u-235/uchmviewer")
include(cmake/get-version.cmake)

include(cmake/use-in.cmake)
option(USE_KF5            "Use KDE Framework 5"                             OFF)
option(USE_WEBENGINE      "Use Qt WebEngine"                                OFF)
option(USE_STATIC_CHMLIB  "Use static link with chm library"                OFF)
option(USE_DEPLOY_RUNTIME "Copy runtime dependencies for deployment"        OFF)
option(USE_GETTEXT        "Use GNU Gettext for translation"                  ON)
option(USE_MACOS_BUNDLE   "Install as macOS bundle"                          ON)
option(IS_VCPKG_BUILD     "Running under vcpkg"                             OFF)
use_in(USE_DBUS           "Use D-Bus integration"                       "Linux")
use_in(USE_MAC_APP        "Use derived QApplication"                   "Darwin")

# Init cmake
list(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake")

# Interface library to collect additional dependencies.
add_library(extra INTERFACE)

# Necessary packages
# libzip
find_package(libzip REQUIRED)

# Prepare to find Qt modules.
set(APP_QT_MIN_VERSION 5.0)
set(APP_QT_MODULES Core Network PrintSupport Widgets Xml)
# Modules for WebKit or WebEngine.
if (${USE_WEBENGINE})
    set(APP_QT_MIN_VERSION 5.9)
    list(APPEND APP_QT_MODULES WebEngineWidgets)
    target_link_libraries(extra INTERFACE Qt::WebEngineWidgets)
    target_compile_definitions(extra INTERFACE USE_WEBENGINE)
else ()
    list(APPEND APP_QT_MODULES WebKit WebKitWidgets)
    target_link_libraries(extra INTERFACE Qt::WebKit Qt::WebKitWidgets)
endif ()

set(APP_QT_VERSION ${APP_QT_MIN_VERSION} CACHE STRING "Preferred Qt version")

# What version of Qt are we working with?
if ($CACHE{APP_QT_VERSION} VERSION_LESS 6)
    find_package(QT $CACHE{APP_QT_VERSION} NAMES Qt5)
endif ()

if (NOT QT_FOUND)
    find_package(QT $CACHE{APP_QT_VERSION} NAMES Qt5 Qt6 REQUIRED)
endif ()


set(QT Qt${QT_VERSION_MAJOR})
# Since Qt 6 QTextCodec moved in the Qt 5 Core Compat module.
if (${QT_VERSION_MAJOR} GREATER_EQUAL 6)
    list(APPEND APP_QT_MODULES Core5Compat)
    target_link_libraries(extra INTERFACE ${QT}::Core5Compat)
endif ()

if (${USE_DBUS})
    list(APPEND APP_QT_MODULES DBus)
    target_link_libraries(extra INTERFACE Qt::DBus)
    target_compile_definitions(extra INTERFACE USE_DBUS)
endif ()

if (${USE_MAC_APP})
    target_compile_definitions(extra INTERFACE USE_MAC_APP)
endif ()

# Now we know all the required Qt modules.
find_package(${QT} REQUIRED ${APP_QT_MODULES})

if (${USE_KF5})
    # Needed by find_package(KF5) below.
    find_package(ECM REQUIRED)
    list(APPEND CMAKE_MODULE_PATH ${ECM_MODULE_PATH})
    find_package(KF5 REQUIRED KDELibs4Support)
    target_link_libraries(extra INTERFACE KF5::KDELibs4Support)
    target_compile_definitions(extra INTERFACE USE_KDE)
endif ()

#
# IS_VCPKG_BUILD is set by the vcpkg portfile.cmake file.  If set, we need to pick up the
# archive files for:
#    bzip2
#    zlib
# as libzip will have been built as an archive not a .so/.dylib
#
if (IS_VCPKG_BUILD)
    find_package(BZip2 REQUIRED)
    find_package(ZLIB 1.1.2 REQUIRED)
endif ()

include(cmake/qt-aliases.cmake)
include(cmake/install-helper.cmake)

add_compile_options(
    $<$<CXX_COMPILER_ID:MSVC>:/W3>
    $<$<NOT:$<CXX_COMPILER_ID:MSVC>>:-Wall>
    $<$<NOT:$<CXX_COMPILER_ID:MSVC>>:-Wextra>)

set(CMAKE_CXX_EXTENSIONS ON)
set(CMAKE_CXX_STANDARD 11)

# Run in those subdirectories
add_subdirectory(lib)
add_subdirectory(src)
add_subdirectory(po)
add_subdirectory(packages)
