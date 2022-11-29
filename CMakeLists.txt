cmake_minimum_required(VERSION 3.14)

project(
  scummvm-ios7-libs
  VERSION 3.0
  LANGUAGES C CXX)

if(CMAKE_SYSTEM_NAME STREQUAL "iOS")
  if(CMAKE_OSX_SYSROOT MATCHES ".*iPhoneSimulator.*")
    message(STATUS "Building for iOS simulator SDK: ${CMAKE_OSX_SYSROOT}")
    set(triple arm64-x86_64-apple-darwin)
    set(target arm64-x86_64-apple-ios-simulator)
  else()
    message(STATUS "Building for iOS SDK: ${CMAKE_OSX_SYSROOT}")
    set(triple aarch64-apple-darwin)
    set(target arm64-apple-ios)
  endif()
elseif(CMAKE_SYSTEM_NAME STREQUAL "tvOS")
    set(target tvos)
  if(CMAKE_OSX_SYSROOT MATCHES ".*appletvsimulator")
    message(STATUS "Building for tvOS simulator SDK: ${CMAKE_OSX_SYSROOT}")
    set(triple arm64-x86_64-apple-tvos${CMAKE_OSX_DEPLOYMENT_TARGET}-simulator)
  else()
    message(STATUS "Building for tvOS SDK: ${CMAKE_OSX_SYSROOT}")
    set(triple arm64-apple-tvos${CMAKE_OSX_DEPLOYMENT_TARGET})
  endif()
else()
  message(ERROR "Unknown system")
endif()

execute_process(COMMAND xcrun -sdk ${CMAKE_OSX_SYSROOT} -find libtool
  OUTPUT_VARIABLE BUILD_LIBTOOL
  ERROR_QUIET
  OUTPUT_STRIP_TRAILING_WHITESPACE)

include(ExternalProject)

ExternalProject_Add(curl
  GIT_REPOSITORY https://github.com/curl/curl.git
  GIT_TAG curl-7_86_0
  PREFIX ${CMAKE_CURRENT_BINARY_DIR}/curl
  CMAKE_ARGS -DBUILD_SHARED_LIBS=OFF -DCURL_ENABLE_SSL=OFF -DCMAKE_INSTALL_PREFIX=${CMAKE_CURRENT_LIST_DIR}/bin
  BUILD_IN_SOURCE TRUE
)

ExternalProject_Add(flac
  GIT_REPOSITORY https://github.com/xiph/flac.git
  GIT_TAG 1.4.2
  PREFIX ${CMAKE_CURRENT_BINARY_DIR}/flac
  CMAKE_ARGS -DBUILD_DOCS=OFF -DBUILD_PROGRAMS=OFF -DBUILD_CXXLIBS=OFF -DBUILD_TESTING=OFF -DWITH_OGG=OFF
    -DINSTALL_MANPAGES=OFF -DINSTALL_PKGCONFIG_MODULES=OFF -DBUILD_EXAMPLES=OFF -DCMAKE_INSTALL_PREFIX=${CMAKE_CURRENT_LIST_DIR}/bin
  BUILD_IN_SOURCE TRUE
)

# fluidsynt stuff

ExternalProject_Add(libffi
  GIT_REPOSITORY https://github.com/libffi/libffi
  GIT_TAG v3.4.4
  PREFIX ${CMAKE_CURRENT_BINARY_DIR}/fluidsynth
  CONFIGURE_COMMAND ./autogen.sh
    COMMAND ./configure
    "--disable-builddir"
    "--disable-shared"
    "--host=${triple}"
    "CFLAGS= -fembed-bitcode -m${SYSTEM_NAME_INT}-version-min=${CMAKE_OSX_DEPLOYMENT_TARGET}"
    "CC=xcrun -sdk ${SYSTEM_NAME_INT} clang -target ${target}"
  BUILD_COMMAND make -C include
    COMMAND make install-pkgconfigDATA install-toolexeclibLTLIBRARIES
  INSTALL_COMMAND make DESTDIR=${CMAKE_CURRENT_LIST_DIR}/bin -C include install
  BUILD_IN_SOURCE TRUE
)

ExternalProject_Add(gettext
  GIT_REPOSITORY https://git.savannah.gnu.org/git/gettext.git
  GIT_TAG v0.21
  PREFIX ${CMAKE_CURRENT_BINARY_DIR}/fluidsynth
  CONFIGURE_COMMAND ./autogen.sh
    COMMAND ./configure
    "--disable-shared"
    "--disable-libasprintf"
    "--disable-java"
    "--disable-c++"
    "--host=${triple}"
    "CFLAGS= -fembed-bitcode -m${SYSTEM_NAME_INT}-version-min=${CMAKE_OSX_DEPLOYMENT_TARGET}"
    "CPPFLAGS = ${CFLAGS}"
    "CC=xcrun -sdk ${SYSTEM_NAME_INT} clang -target ${target}"
    "CPP=xcrun -sdk ${SYSTEM_NAME_INT} clang -target ${target}"
  BUILD_COMMAND make -C gettext-runtime/intl
  INSTALL_COMMAND make DESTDIR=${CMAKE_CURRENT_LIST_DIR}/bin -C gettext-runtime/intl install
  BUILD_IN_SOURCE TRUE
)

find_program (MESON_EXECUTABLE meson DOC "Meson build system")
#find_program (NINJA_EXECUTABLE ninja DOC "Ninja build tool")

ExternalProject_Add(glib
  GIT_REPOSITORY https://github.com/GNOME/glib.git
  GIT_TAG 2.75.0
  PREFIX ${CMAKE_CURRENT_BINARY_DIR}/fluidsynth
  CONFIGURE_COMMAND ${CMAKE_COMMAND} -E env C=${CMAKE_C_COMPILER}
    ${CMAKE_COMMAND} -E env CXX=${CMAKE_CXX_COMPILER}
    ${CMAKE_COMMAND} -E env OBJC=${CMAKE_C_COMPILER}
    ${CMAKE_COMMAND} -E env CFLAGS=${CMAKE_C_FLAGS}
    ${CMAKE_COMMAND} -E env CPPFLAGS=${CMAKE_C_FLAGS}
    ${CMAKE_COMMAND} -E env CXXFLAGS=${CMAKE_C_FLAGS}
    ${CMAKE_COMMAND} -E env LDFLAGS=${CMAKE_C_FLAGS}
    python ${CMAKE_CURRENT_BINARY_DIR}/meson-crossgen --system darwin --cpu aarch64 --sysroot ${CMAKE_OSX_SYSROOT} -o ${CMAKE_CURRENT_BINARY_DIR}/cross.ini
    COMMAND ${MESON_EXECUTABLE} setup --cross-file ${CMAKE_CURRENT_BINARY_DIR}/cross.ini _build
  BUILD_COMMAND "${MESON_EXECUTABLE} compile --buildtype ${CMAKE_BUILD_TYPE} --default-library static -C _build"
  INSTALL_COMMAND ""
  BUILD_IN_SOURCE TRUE
)
