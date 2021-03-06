# cmake file for ALT Sisyphus
# see also https://github.com/Francesco149/loli-overlay/blob/master/net-im/telegram-desktop/files/Telegram.cmake

cmake_minimum_required(VERSION 3.4)

project(TelegramDesktop)

set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/gyp ${CMAKE_SOURCE_DIR}/cmake ${CMAKE_MODULE_PATH})

set(CMAKE_CXX_STANDARD 17)

set(CMAKE_INCLUDE_CURRENT_DIR ON)
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTORCC ON)


option(GTK "GTK+ (currently, v3) Integration" ON)

# FIXME: doesn't work for DBus, we need add QtDbus below and fix linking
find_package(Qt5 REQUIRED COMPONENTS Core Gui Widgets Network DBus)

get_target_property(QTCORE_INCLUDE_DIRS Qt5::Core INTERFACE_INCLUDE_DIRECTORIES)
list(GET QTCORE_INCLUDE_DIRS 0 QT_INCLUDE_DIR)

foreach(__qt_module IN ITEMS QtCore QtGui QtDBus)
    list(APPEND QT_PRIVATE_INCLUDE_DIRS
        ${QT_INCLUDE_DIR}/${__qt_module})
    list(APPEND QT_PRIVATE_INCLUDE_DIRS
        ${QT_INCLUDE_DIR}/${__qt_module}/${Qt5_VERSION})
    list(APPEND QT_PRIVATE_INCLUDE_DIRS
        ${QT_INCLUDE_DIR}/${__qt_module}/${Qt5_VERSION}/${__qt_module})
endforeach()
message(STATUS "Using Qt private include directories: ${QT_PRIVATE_INCLUDE_DIRS}")

find_package(PkgConfig REQUIRED)

pkg_check_modules(FFMPEG REQUIRED libavformat libavcodec libswresample libswscale libavutil)
pkg_check_modules(TGVOIP REQUIRED libtgvoip)
pkg_check_modules(LIBCRL REQUIRED libcrl)
pkg_check_modules(LIBDRM REQUIRED libdrm)
pkg_check_modules(LIBVA REQUIRED libva-x11 libva-drm libva)
pkg_check_modules(MINIZIP REQUIRED minizip)
pkg_check_modules(LIBLZ4 REQUIRED liblz4)

if(GTK)
  pkg_check_modules(GTK3 REQUIRED gtk+-3.0)
  #pkg_check_modules(DEE REQUIRED dee-1.0)
  pkg_check_modules(APPINDICATOR REQUIRED appindicator3-0.1)
else()
  set(GTK_INTEGRATION TDESKTOP_DISABLE_GTK_INTEGRATION)
endif()

find_package(ZLIB REQUIRED)

find_package(Threads REQUIRED)
find_package(OpenAL REQUIRED)
find_package(OpenSSL REQUIRED)
find_package(LibLZMA REQUIRED)
find_package(X11 REQUIRED)
find_package(rlottie REQUIRED)

set(GSL_INCLUDE_DIR /usr/include/libmicrosoft-gsl)
set(VARIANT_INCLUDE_DIR /usr/include/libvariant)
set(RANGEV3_INCLUDE_DIR /usr/include/range-v3)

set(EMOJI_SUGGESTIONS_INCLUDE_DIR ${CMAKE_SOURCE_DIR}/ThirdParty/emoji_suggestions)
set(QTLOTTIE_INCLUDE_DIR ${CMAKE_SOURCE_DIR}/ThirdParty/qtlottie/src/bodymovin ${CMAKE_SOURCE_DIR}/ThirdParty/qtlottie/src/imports)

set(TELEGRAM_SOURCES_DIR ${CMAKE_SOURCE_DIR}/SourceFiles)
set(TELEGRAM_RESOURCES_DIR ${CMAKE_SOURCE_DIR}/Resources)
include_directories(${TELEGRAM_SOURCES_DIR})

set(GENERATED_DIR ${CMAKE_BINARY_DIR}/generated)
file(MAKE_DIRECTORY ${GENERATED_DIR})

include(ExternalProject)

#find_package(Breakpad REQUIRED)

include(PrecompiledHeader)

include(TelegramCodegenTools)
include(TelegramCodegen)

# use list of source files generated from Telegram/gyp/telegram_sources.txt
include(TelegramSourceFiles)

# add precompiled header after tool build
set(CMAKE_AUTOMOC_MOC_OPTIONS ${CMAKE_AUTOMOC_MOC_OPTIONS} "-b" "Telegram_pch/stdafx.h")

add_compile_definitions(
    Q_OS_LINUX64
    __STDC_FORMAT_MACROS
    # AppID for Basealt build
    # got from https://core.telegram.org/api/obtaining_api_id
    TDESKTOP_API_ID=182015
    TDESKTOP_API_HASH=bb6c3f8fffd8fe6804fc5131a08e1c44
)


set(QRC_FILES
    Resources/qrc/telegram.qrc
    Resources/qrc/telegram_emoji_1.qrc
    Resources/qrc/telegram_emoji_2.qrc
    Resources/qrc/telegram_emoji_3.qrc
    Resources/qrc/telegram_emoji_4.qrc
    Resources/qrc/telegram_emoji_5.qrc
    Resources/qrc/telegram_sounds.qrc
    # This only disables system plugin search path
    # We do not want this behavior for system build
    #Resources/qrc/telegram_linux.qrc
)

# We do not want to include Qt plugins statically
list(REMOVE_ITEM SOURCE_FILES SourceFiles/qt_static_plugins.cpp)
list(APPEND SOURCE_FILES SourceFiles/qt_functions.cpp)

# lib_base, lib_export, lib_storage
file(GLOB_RECURSE LIB_SOURCE_FILES
    SourceFiles/base/*.cpp SourceFiles/base/*.h
    SourceFiles/export/*.cpp SourceFiles/export/*.h
    SourceFiles/storage/*.cpp SourceFiles/storage/*.h
)

# No unit-tests in non-CI builds plz
file(GLOB_RECURSE NO_TESTS SourceFiles/*/*tests*.cpp)
list(REMOVE_ITEM LIB_SOURCE_FILES ${NO_TESTS})

list(REMOVE_ITEM LIB_SOURCE_FILES ${CMAKE_SOURCE_DIR}/SourceFiles/storage/storage_feed_messages.h)
list(REMOVE_ITEM LIB_SOURCE_FILES ${CMAKE_SOURCE_DIR}/SourceFiles/storage/storage_feed_messages.cpp)
list(REMOVE_ITEM LIB_SOURCE_FILES ${CMAKE_SOURCE_DIR}/SourceFiles/storage/storage_clear_legacy_win.cpp)
list(REMOVE_ITEM LIB_SOURCE_FILES ${CMAKE_SOURCE_DIR}/SourceFiles/storage/storage_file_lock_win.cpp)

file(GLOB_RECURSE EMOJI_SOURCE_FILES
    ${EMOJI_SUGGESTIONS_INCLUDE_DIR}/*.cpp ${EMOJI_SUGGESTIONS_INCLUDE_DIR}/*.h
)

add_executable(Telegram WIN32 ${QRC_FILES} ${LIB_SOURCE_FILES} ${SOURCE_FILES} ${EMOJI_SOURCE_FILES})

# TODO: improve (need just pkg-config --static)
if(FFMPEG_STATIC_LIBRARY_DIRS)
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -L${FFMPEG_STATIC_LIBRARY_DIRS}")
endif()

target_link_libraries(Telegram
    Qt5::Widgets
    Qt5::Network
    Qt5::DBus
    Threads::Threads
    OpenSSL::SSL
    OpenSSL::Crypto
#    breakpad
    ${FFMPEG_LIBRARIES}
# was missed in libavutil.pc, but links only when here
    -lvdpau
    -lxxhash
    ${LIBVA_LIBRARIES}
    ${LIBDRM_LIBRARIES}
    ${ZLIB_LIBRARY_RELEASE}
    ${GTK3_LIBRARIES}
    ${LIBCRL_LIBRARIES}
    ${TGVOIP_LIBRARIES}
    ${MINIZIP_LIBRARIES}
    ${OPENAL_LIBRARY}
    ${APPINDICATOR_LIBRARIES}
    ${DEE_LIBRARIES}
    ${LIBLZMA_LIBRARIES}
    ${rlottie_LIBRARIES}
    ${LIBLZ4_LIBRARIES}
    ${X11_X11_LIB}
    )

target_include_directories(Telegram PUBLIC
    ${QT_PRIVATE_INCLUDE_DIRS}
    ${FFMPEG_INCLUDE_DIRS}
    ${LIBVA_INCLUDE_DIRS}
    ${LIBDRM_INCLUDE_DIRS}
    ${ZLIB_INCLUDE_DIR}
    ${GTK3_INCLUDE_DIRS}
    ${LIBCRL_INCLUDE_DIRS}
    ${TGVOIP_INCLUDE_DIRS}
    ${GSL_INCLUDE_DIR}
    ${EMOJI_SUGGESTIONS_INCLUDE_DIR}
    ${VARIANT_INCLUDE_DIR}
    ${RANGEV3_INCLUDE_DIR}
    ${MINIZIP_INCLUDE_DIRS}
    ${OPENAL_INCLUDE_DIR}
    ${APPINDICATOR_INCLUDE_DIRS}
    ${DEE_INCLUDE_DIRS}
    ${LIBLZMA_INCLUDE_DIRS}
    ${QTLOTTIE_INCLUDE_DIR}
    ${LIBLZ4_INCLUDE_DIRS}
    ${GENERATED_DIR})

target_sources(Telegram PRIVATE ${TELEGRAM_GENERATED_SOURCES})
# we need codegenerators' output for precompiled headers needed for Telegram target
add_dependencies(Telegram telegram_codegen)

add_precompiled_header(Telegram SourceFiles/stdafx.h)

target_compile_definitions(Telegram PUBLIC
    TDESKTOP_DISABLE_UNITY_INTEGRATION
    ## Win only openal ext:
    TDESKTOP_DISABLE_OPENAL_EFFECTS
    TDESKTOP_DISABLE_DESKTOP_FILE_GENERATION
    TDESKTOP_DISABLE_REGISTER_CUSTOM_SCHEME
    TDESKTOP_DISABLE_CRASH_REPORTS
    TDESKTOP_DISABLE_AUTOUPDATE
    TDESKTOP_DISABLE_GTK_INTEGRATION
)

# TODO: does not work yet
# https://cmake.org/cmake/help/v3.4/prop_tgt/AUTOMOC_MOC_OPTIONS.html
#set_target_properties(Telegram PROPERTIES
#    AUTOMOC ON
#    AUTOMOC_MOC_OPTIONS "-b Telegram_pch/stdafx.h"
#    )

install(TARGETS Telegram RUNTIME DESTINATION bin)
install(PROGRAMS ${CMAKE_SOURCE_DIR}/../lib/xdg/telegram-desktop.desktop DESTINATION share/applications)
