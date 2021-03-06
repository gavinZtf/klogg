cmake_minimum_required(VERSION 3.2)

project(klogg VERSION 17.12.0)

set (CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake")

include(ucm)

set(BUILD_VERSION "" CACHE STRING "build version")
set(BUILD_NUMBER "" CACHE STRING "build number")

string(TIMESTAMP PRODUCT_CURRENT_YEAR "%Y")

set(COMPANY "Anton Filimonov")
set(COPYRIGHT "Copyright (c) ${PRODUCT_CURRENT_YEAR} Nicolas Bonnefon, Anton Filimonov and other contributors.")
set(IDENTIFIER "com.github.variar.klogg")
set(PROJECT_DESCRIPTION "${PROJECT_NAME} log viewer")

if (WIN32)
    set(ICON_NAME glogg48.ico)
elseif(APPLE)
    set(ICON_NAME glogg.icns)
endif()

if (NOT ${ICON_NAME} STREQUAL "")
    set(ICON_FILE ${CMAKE_CURRENT_SOURCE_DIR}/Resources/${ICON_NAME})
endif()

include(prepare_version)

set(out_dir "${CMAKE_BINARY_DIR}/output")
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${out_dir}")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${out_dir}")
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${out_dir}")

set_property(GLOBAL PROPERTY USE_FOLDERS ON)

set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

find_package(Qt5Core REQUIRED)
find_package(Qt5Widgets REQUIRED)
find_package(Qt5Concurrent REQUIRED)

set(CMAKE_AUTOMOC ON)
set(CMAKE_INCLUDE_CURRENT_DIR ON)

macro(remove_release_flag flag)
    string(REPLACE "${flag}" "" CMAKE_C_FLAGS_RELEASE "${CMAKE_C_FLAGS_RELEASE}")
    string(REPLACE "${flag}" "" CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE}")
endmacro()

# warnings and optimization
if(NOT MSVC)
    ucm_add_flags(-O2 -g)
    ucm_add_flags(-Werror -Wall -Wextra -Wnarrowing -Wshadow -Wnon-virtual-dtor -Wpedantic)
    remove_release_flag("-O3")
else()
    ucm_add_flags(/O2)

    IF (CMAKE_CXX_FLAGS MATCHES "/W[0-4]")
        STRING(REGEX REPLACE "/W[0-4]" "/W4" CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS}")
    ELSE (CMAKE_CXX_FLAGS MATCHES "/W[0-4]")
        SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /W4")
    ENDIF (CMAKE_CXX_FLAGS MATCHES "/W[0-4]")

    IF (CMAKE_C_FLAGS MATCHES "/W[0-4]")
        STRING(REGEX REPLACE "/W[0-4]" "/W4" CMAKE_C_FLAGS "${CMAKE_C_FLAGS}")
    ELSE (CMAKE_C_FLAGS MATCHES "/W[0-4]")
        SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /W4")
    ENDIF (CMAKE_C_FLAGS MATCHES "/W[0-4]")

    ucm_add_flags(/wd4127)

endif()

ucm_print_flags()

if (UNIX)
    option(GPROF "Build for gprof" OFF)
    if (${GPROF})
        ucm_add_flags(-pg -g)
        ucm_add_linker_flags(-pg)
    endif()

    option(PREFER_DBUS_IPC "Prefer dbus IPC if available" ON)
endif()

if (APPLE)
    ucm_add_flags(CXX -stdlib=libc++)
    set(CMAKE_OSX_DEPLOYMENT_TARGET 10.7)
endif()

if (WIN32)
    add_definitions(-DNOMINMAX)
endif()

option(PERF "Build for performance test" OFF)
if(${PERF})
    add_definitions(-DGLOGG_PERF_MEASURE_FPS)
endif()

add_custom_target(ci_config SOURCES
    ${CMAKE_CURRENT_SOURCE_DIR}/.travis.yml
    ${CMAKE_CURRENT_SOURCE_DIR}/appveyor.yml
    ${CMAKE_CURRENT_SOURCE_DIR}/scripts/linux/make_appimage.sh
    ${CMAKE_CURRENT_SOURCE_DIR}/scripts/osx/install_deps.sh
    ${CMAKE_CURRENT_SOURCE_DIR}/scripts/osx/prepare_bundle.sh
)

add_custom_target(installation SOURCES
    ${CMAKE_CURRENT_SOURCE_DIR}/klogg.nsi
    ${CMAKE_CURRENT_SOURCE_DIR}/klogg.desktop
    ${CMAKE_CURRENT_SOURCE_DIR}/cmake/MacOSXBundleInfo.plist.in
)

add_custom_target(docs SOURCES
    ${CMAKE_CURRENT_SOURCE_DIR}/README.md
    ${CMAKE_CURRENT_SOURCE_DIR}/COPYING
)

add_subdirectory(3rdparty)
add_subdirectory(src)

option(BUILD_TESTS "Build tests" ON)
if (${BUILD_TESTS})
    enable_testing()

    add_test(
        NAME klogg_smoke
        COMMAND klogg -platform offscreen -v
    )

    add_subdirectory(tests)   
endif()

# Installation and packaging
if(UNIX)
    include(InstallRequiredSystemLibraries)

    SET(CPACK_OUTPUT_FILE_PREFIX "packages")
    SET(CPACK_PACKAGE_DESCRIPTION_SUMMARY ${PROJECT_DESCRIPTION})
    SET(CPACK_PACKAGE_VENDOR ${COMPANY})
    SET(CPACK_PACKAGE_CONTACT "anton.filimonov@gmail.com")
    SET(CPACK_PACKAGE_DESCRIPTION_FILE "${CMAKE_CURRENT_SOURCE_DIR}/README.md")
    SET(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/COPYING")
    SET(CPACK_PACKAGE_VERSION "${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR}.${PROJECT_VERSION_PATCH}-r${PROJECT_VERSION_TWEAK}")
    SET(CPACK_STRIP_FILES "bin/klogg")

    if (APPLE)
        set(CPACK_GENERATOR "DragNDrop")
        set(CPACK_DMG_FORMAT "UDBZ")
        set(CPACK_DMG_VOLUME_NAME "${PROJECT_NAME}")
        set(CPACK_SYSTEM_NAME "OSX")
        set(CPACK_PACKAGE_ICON "${ICON_FILE}")
    else()
        install(FILES
            ${CMAKE_CURRENT_SOURCE_DIR}/src/app/images/hicolor/16x16/klogg.png
            DESTINATION share/icons/hicolor/16x16/apps
        )

        install(FILES
            ${CMAKE_CURRENT_SOURCE_DIR}/src/app/images/hicolor/32x32/klogg.png
            DESTINATION share/icons/hicolor/32x32/apps
        )

        install(FILES
            ${CMAKE_CURRENT_SOURCE_DIR}/src/app/images/hicolor/scalable/klogg.svg
            DESTINATION share/icons/hicolor/scalable/apps
        )

        install(FILES
            ${CMAKE_CURRENT_SOURCE_DIR}/COPYING
            ${CMAKE_CURRENT_SOURCE_DIR}/README.md
            DESTINATION share/doc/klogg)

        install(FILES
            ${CMAKE_CURRENT_SOURCE_DIR}/klogg.desktop
            DESTINATION share/applications
        )

        # linux
        IF(CPACK_GENERATOR STREQUAL "RPM")
            set(CPACK_RPM_PACKAGE_LICENSE "GPLv3")
            set(CPACK_RPM_PACKAGE_REQUIRES "qt5-qtbase")
            set(CPACK_RPM_PACKAGE_URL "http://github.com/variar/klogg")
        ELSE()
            set(CPACK_GENERATOR DEB)
            set(CPACK_DEBIAN_PACKAGE_SECTION "utils")
            set(CPACK_DEBIAN_PACKAGE_HOMEPAGE "http://github.com/variar/klogg")
            set(CPACK_DEBIAN_PACKAGE_DEPENDS "libqt5dbus5, libqt5concurrent5, libqt5widgets5")
        ENDIF()
    endif()
    INCLUDE(CPack)
endif()




