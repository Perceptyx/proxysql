cmake_minimum_required(VERSION 3.10)

# set the project name
project(proxysql VERSION 2.0.11)

include(ExternalProject)
MESSAGE(STATUS "SYSTEM processor: ${CMAKE_SYSTEM_PROCESSOR}")
SET(CPACK_PACKAGE_VENDOR "sysown")
SET(CPACK_PACKAGE_DESCRIPTION "High-performance MySQL proxy with a GPL license")
SET(CPACK_PACKAGE_NAME "proxysql")
STRING(TOLOWER ${CMAKE_SYSTEM_NAME} system_name)
SET(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/LICENSE")
SET(CPACK_PACKAGE_DESCRIPTION_FILE "${CMAKE_CURRENT_SOURCE_DIR}/README.md")
SET(CPACK_SOURCE_PACKAGE_FILE_NAME "proxysql${CPACK_PACKAGE_VERSION}-${PACKAGE_STATUS_SUFFIX}-src")
SET(CPACK_PACKAGE_FILE_NAME "proxysql-${CPACK_PACKAGE_VERSION}-${PACKAGE_STATUS_SUFFIX}-${system_name}-${CMAKE_SYSTEM_PROCESSOR}")
SET(CPACK_PACKAGE_VERSION_MAJOR 2)
SET(CPACK_PACKAGE_VERSION_MINOR 0)
SET(CPACK_PACKAGE_VERSION_PATCH 11)
SET(CPACK_PACKAGE_HOMEPAGE_URL https://www.proxysql.com)
#SET(CPACK_PROJECT_CONFIG_FILE ${CMAKE_CURRENT_SOURCE_DIR}/src/proxysql.cfg)
SET(CPACK_SOURCE_GENERATOR "RPM")

install(PROGRAMS    ${CMAKE_CURRENT_BINARY_DIR}/proxysql
        TYPE        BIN
)

install(FILES       ${CMAKE_CURRENT_SOURCE_DIR}/src/proxysql.cfg
        TYPE        SYSCONF
)

include(CPack)

# specify the C++ standard
set(CMAKE_CXX_STANDARD 11)
set(CMAKE_CXX_STANDARD_REQUIRED True)

find_program(MAKE_EXE NAMES gmake nmake make)

# For the GIT_STATUS -D
find_package (Git)
if (GIT_FOUND)
  execute_process(COMMAND ${GIT_EXECUTABLE} describe --long
      WORKING_DIRECTORY "${CMAKE_SOURCE_DIR}"
      RESULT_VARIABLE result
      OUTPUT_VARIABLE repo_git_version
      ERROR_QUIET
      OUTPUT_STRIP_TRAILING_WHITESPACE)

  message(STATUS "Using GIT_VERSION=${repo_git_version}")
else()
  message(WARNING "No GIT_VERSION will be added to resulting binary.")
  set(repo_git_version "undefined")
endif()

add_definitions(-DGITVERSION="$(repo_git_version)")

# Require GNU TLS for building the libmicrohttpd library
find_package(GnuTLS REQUIRED)
if(GNUTLS_FOUND)
    message(STATUS "Library 'GnuTLS' found at ${GNUTLS_LIBRARIES}")
else()
    message(FATAL_ERROR "Library 'GnuTLS' not found! Install it vie e.g. 'brew install gnutls' or 'dnf install gnutls-devel'")
endif()

# Libinjection uses python for generating stuff
# code is written in py2 syntax
find_package (Python REQUIRED COMPONENTS Interpreter)

SET (LIBINJECTION_PATCH_COMMAND "")

if (Python_VERSION_MAJOR EQUAL "3")
  message(STATUS "Will patch libinjection for python3 support")
  SET (LIBINJECTION_PATCH_COMMAND 2to3 -w src/fingerprints2sqli.py src/make_parens.py src/sqlparse2c.py src/sqlparse_map.py)
endif()

# Typically you don't care so much for a third party library's tests to be
# run from your own project's code.
set(JSON_BuildTests OFF CACHE INTERNAL "")

# If you only include this third party in PRIVATE source files, you do not
# need to install it when your main project gets installed.
# set(JSON_Install OFF CACHE INTERNAL "")

# Don't use include(nlohmann_json/CMakeLists.txt) since that carries with it
# unintended consequences that will break the build.  It's generally
# discouraged (although not necessarily well documented as such) to use
# include(...) for pulling in other CMake projects anyways.
add_subdirectory("${PROJECT_SOURCE_DIR}/external/nlohmann_json")

SET(PATCHES_DIR ${CMAKE_CURRENT_SOURCE_DIR}/patches)

ExternalProject_Add(libconfig_ext
	PREFIX						deps
  URL               https://hyperrealm.github.io/libconfig/dist/libconfig-1.7.2.tar.gz
  URL_HASH          SHA1=1da1e7f4e0e376582a8414d2be1acc9dbf6649b5
  CONFIGURE_COMMAND ./configure
  BUILD_IN_SOURCE 1
  BUILD_COMMAND     ${MAKE_EXE}
  UPDATE_COMMAND ""
  INSTALL_COMMAND ""
)

SET(LIBCONFIG_INCLUDE_DIR ${CMAKE_CURRENT_BINARY_DIR}/deps/src/libconfig_ext/lib)
SET(LIBCONFIG_LIB_DIR ${CMAKE_CURRENT_BINARY_DIR}/deps/src/libconfig_ext/lib/.libs)
add_library(libconfig++ STATIC IMPORTED)
set_target_properties(libconfig++ PROPERTIES IMPORTED_LOCATION ${LIBCONFIG_LIB_DIR}/libconfig++.a)

ExternalProject_Add(libdaemon_ext
	PREFIX						deps
  URL               http://0pointer.de/lennart/projects/libdaemon/libdaemon-0.14.tar.gz
  URL_HASH          SHA1=78a4db58cf3a7a8906c35592434e37680ca83b8f
  CONFIGURE_COMMAND ./configure --disable-examples
  BUILD_IN_SOURCE 1
  BUILD_COMMAND     ${MAKE_EXE}
  UPDATE_COMMAND ""
  INSTALL_COMMAND ""
)

SET(LIBDAEMON_INCLUDE_DIR ${CMAKE_CURRENT_BINARY_DIR}/deps/src/libdaemon_ext)
SET(LIBDAEMON_LIB_DIR ${CMAKE_CURRENT_BINARY_DIR}/deps/src/libdaemon_ext/libdaemon/.libs)
add_library(libdaemon STATIC IMPORTED)
set_target_properties(libdaemon PROPERTIES IMPORTED_LOCATION ${LIBDAEMON_LIB_DIR}/libdaemon.a)


ExternalProject_Add(jemalloc_ext
	PREFIX						deps
  URL               https://github.com/jemalloc/jemalloc/releases/download/5.2.1/jemalloc-5.2.1.tar.bz2
  URL_HASH          SHA1=9e06b5cc57fd185379d007696da153893cf73e30
  CONFIGURE_COMMAND ./configure
  BUILD_IN_SOURCE 1
  BUILD_COMMAND     ${MAKE_EXE}
  UPDATE_COMMAND ""
  INSTALL_COMMAND ""
)

SET(JEMALLOC_INCLUDE_DIR ${CMAKE_CURRENT_BINARY_DIR}/deps/src/jemalloc_ext/include/jemalloc)
SET(JEMALLOC_LIB_DIR ${CMAKE_CURRENT_BINARY_DIR}/deps/src/jemalloc_ext/lib)
add_library(jemalloc STATIC IMPORTED)
set_target_properties(jemalloc PROPERTIES IMPORTED_LOCATION ${JEMALLOC_LIB_DIR}/libjemalloc.a)

ExternalProject_Add(cityhash_ext
	PREFIX						deps
  URL               https://github.com/google/cityhash/archive/master.zip
  URL_HASH          SHA1=6c527dfc36ba12157113163e7666a3d501a77077
  CONFIGURE_COMMAND ./configure
  BUILD_IN_SOURCE 1
  BUILD_COMMAND     ${MAKE_EXE}
  UPDATE_COMMAND ""
  INSTALL_COMMAND ""
)

SET(CITYHASH_INCLUDE_DIR ${CMAKE_CURRENT_BINARY_DIR}/deps/src/cityhash_ext/include)
SET(CITYHASH_LIB_DIR ${CMAKE_CURRENT_BINARY_DIR}/deps/src/cityhash_ext/src/.libs)
add_library(cityhash STATIC IMPORTED)
set_target_properties(cityhash PROPERTIES IMPORTED_LOCATION ${CITYHASH_LIB_DIR}/libcityhash.a)

ExternalProject_Add(libinjection_ext
	PREFIX						deps
  URL               https://github.com/client9/libinjection/archive/v3.10.0.tar.gz
  URL_HASH          SHA1=74a2b44f8fe946f82df7f471258190230cd6d3a4
  CONFIGURE_COMMAND ""
  BUILD_IN_SOURCE 1
  BUILD_COMMAND     ${MAKE_EXE} -C src all
  UPDATE_COMMAND    ""
  INSTALL_COMMAND   ""
  PATCH_COMMAND     ${LIBINJECTION_PATCH_COMMAND}
)

SET(LIBINJECTION_LIB_DIR ${CMAKE_CURRENT_BINARY_DIR}/deps/src/libinjection_ext/src)
SET(LIBINJECTION_INCLUDE_DIR ${CMAKE_CURRENT_BINARY_DIR}/deps/src/libinjection_ext/src/)
add_library(libinjection STATIC IMPORTED)
set_target_properties(libinjection PROPERTIES IMPORTED_LOCATION ${LIBINJECTION_LIB_DIR}/libinjection.a)

ExternalProject_Add(openssl_ext
	PREFIX						deps
  URL               https://github.com/openssl/openssl/archive/OpenSSL_1_1_1d.tar.gz
  URL_HASH          SHA1=df0ee4811c87c209ebadb4e6b203d1e560d00f9a
  CONFIGURE_COMMAND ./config no-ssl3
  BUILD_IN_SOURCE 1
  BUILD_COMMAND     ${MAKE_EXE}
  UPDATE_COMMAND    ""
  INSTALL_COMMAND   ""
  PATCH_COMMAND			""
)

SET(OPENSSL_DIR ${CMAKE_CURRENT_BINARY_DIR}/deps/src/openssl_ext)
SET(OPENSSL_LIB_DIR ${CMAKE_CURRENT_BINARY_DIR}/deps/src/openssl_ext)
SET(OPENSSL_INCLUDE_DIR ${CMAKE_CURRENT_BINARY_DIR}/deps/src/openssl_ext/include)
add_library(ssl STATIC IMPORTED)
set_target_properties(ssl PROPERTIES IMPORTED_LOCATION ${OPENSSL_DIR}/libssl.a)
add_library(crypto STATIC IMPORTED)
set_target_properties(crypto PROPERTIES IMPORTED_LOCATION ${OPENSSL_DIR}/libcrypto.a)

ExternalProject_Add(curl_ext
	PREFIX						deps
  URL               https://github.com/curl/curl/releases/download/curl-7_69_1/curl-7.69.1.tar.bz2
  URL_HASH          SHA1=f330c29aef026faff98bc0d712cbbe3c00d5c06e
    CONFIGURE_COMMAND ./configure --disable-debug --disable-ftp --disable-ldap --disable-ldaps --disable-rtsp
                        --disable-dict --disable-telnet --disable-tftp --disable-pop3 --disable-imap
                        --disable-smb --disable-smtp --disable-gopher --disable-manual --disable-ipv6 --disable-sspi
                        --disable-crypto-auth --disable-ntlm-wb --disable-tls-srp --without-nghttp2 --without-libidn2
                        --without-depssh2 --without-brotli --disable-shared --with-ssl=${OPENSSL_DIR}
  BUILD_IN_SOURCE 1
  BUILD_COMMAND     ${MAKE_EXE}
  UPDATE_COMMAND    ""
  INSTALL_COMMAND   ""
  PATCH_COMMAND			""
)

SET(CURL_LIB_DIR ${CMAKE_CURRENT_BINARY_DIR}/deps/src/curl_ext/lib/.libs)
SET(CURL_INCLUDE_DIR ${CMAKE_CURRENT_BINARY_DIR}/deps/src/curl_ext/include)
add_library(curl STATIC IMPORTED)
set_target_properties(curl PROPERTIES IMPORTED_LOCATION ${CURL_LIB_DIR}/libcurl.a)

ExternalProject_Add(libev_ext
	PREFIX						deps
  URL               http://dist.schmorp.de/libev/Attic/libev-4.24.tar.gz
  URL_HASH          SHA1=436dd8eff00a45f8805b8cacfe4dd3bd993caedb
  CONFIGURE_COMMAND "./configure"
  BUILD_IN_SOURCE 1
  BUILD_COMMAND     ${MAKE_EXE}
  UPDATE_COMMAND    ""
  INSTALL_COMMAND   ""
  PATCH_COMMAND			""
)

SET(LIBEV_LIB_DIR ${CMAKE_CURRENT_BINARY_DIR}/deps/src/libev_ext/.libs)
SET(LIBEV_INCLUDE_DIR ${CMAKE_CURRENT_BINARY_DIR}/deps/src/libev_ext/include)
add_library(libev STATIC IMPORTED)
set_target_properties(libev PROPERTIES IMPORTED_LOCATION ${LIBEV_LIB_DIR}/libev.a)

ExternalProject_Add(libre2_ext
	PREFIX						deps
  URL               https://github.com/google/re2/archive/2018-07-01.tar.gz
  URL_HASH          SHA1=a66685c9876fc5d3412e0134feb45e353c1c4b3b
  BUILD_IN_SOURCE 1
  UPDATE_COMMAND    ""
  INSTALL_COMMAND   ""
  PATCH_COMMAND			""
)

SET(LIBRE2_LIB_DIR ${CMAKE_CURRENT_BINARY_DIR}/deps/src/libre2_ext)
SET(LIBRE2_INCLUDE_DIR ${CMAKE_CURRENT_BINARY_DIR}/deps/src/libre2_ext)
add_library(libre2 STATIC IMPORTED)
set_target_properties(libre2 PROPERTIES IMPORTED_LOCATION ${LIBRE2_LIB_DIR}/libre2.a)

ExternalProject_Add(libmicrohttpd_ext
	PREFIX						deps
  URL               https://ftp.gnu.org/gnu/libmicrohttpd/libmicrohttpd-0.9.68.tar.gz
  URL_HASH          SHA1=ed1a586727673de25c27ff294700620a39671024
  CONFIGURE_COMMAND ./configure --enable-https
  BUILD_IN_SOURCE 1
  BUILD_COMMAND     ${MAKE_EXE}
  UPDATE_COMMAND    ""
  INSTALL_COMMAND   ""
  PATCH_COMMAND			""
)

SET(LIBMICROHTTPD_LIB_DIR ${CMAKE_CURRENT_BINARY_DIR}/deps/src/libmicrohttpd_ext/src/microhttpd/.libs)
SET(LIBMICROHTTPD_INCLUDE_DIR ${CMAKE_CURRENT_BINARY_DIR}/deps/src/libmicrohttpd_ext/src/include)
# We need both dirs for the libproxysql
SET(LIBMICROHTTPD_INCLUDE_DIRS ${CMAKE_CURRENT_BINARY_DIR}/deps/src/libmicrohttpd_ext/src/include ${CMAKE_CURRENT_BINARY_DIR}/deps/src/libmicrohttpd_ext)
add_library(microhttpd STATIC IMPORTED)
set_target_properties(microhttpd PROPERTIES IMPORTED_LOCATION ${LIBMICROHTTPD_LIB_DIR}/libmicrohttpd.a)
#SET(LIBMICROHTTP_CPPFLAGS "")

#FOREACH(ITEM ${LIBMICROHTTPD_INCLUDE_DIR})
#  LIST(APPEND LIBMICROHTTP_CPPFLAGS "-I${ITEM}")
#ENDFOREACH(ITEM)

#message(STATUS "Using LIBMICROHTTP_CPPFLAGS ${LIBMICROHTTP_CPPFLAGS}")

ExternalProject_Add(libhttpserver_ext
	PREFIX						deps
  URL        				https://github.com/etr/libhttpserver/archive/master.zip
  URL_HASH          SHA1=c5564db0623e1f202b3c496a451b9fb474af4109
  CONFIGURE_COMMAND ./bootstrap && mkdir build && cd build &&
                      CPPFLAGS=-I${LIBMICROHTTPD_INCLUDE_DIR} LIBS=-lmicrohttpd LDFLAGS=-L${LIBMICROHTTPD_LIB_DIR}
                      LD_LIBRARY_PATH=${LIBMICROHTTPD_LIB_DIR} ../configure --enable-static --disable-shared
                        --disable-doxygen-doc --disable-doxygen-dot
                        --disable-doxygen-man --disable-doxygen-html
  BUILD_IN_SOURCE 1
  BUILD_COMMAND     ${MAKE_EXE} -C build
  UPDATE_COMMAND    ""
  INSTALL_COMMAND   ""
  PATCH_COMMAND			""
)

SET(LIBHTTPSERVER_LIB_DIR ${CMAKE_CURRENT_BINARY_DIR}/deps/src/libhttpserver_ext/build/src/.libs)
SET(LIBHTTPSERVER_INCLUDE_DIR ${CMAKE_CURRENT_BINARY_DIR}/deps/src/libhttpserver_ext/src)
add_library(libhttpserver STATIC IMPORTED)
set_target_properties(libhttpserver PROPERTIES IMPORTED_LOCATION ${LIBHTTPSERVER_LIB_DIR}/libhttpserver.a)
add_dependencies(libhttpserver libmicrohttpd)

ExternalProject_Add(lz4_ext
	PREFIX						deps
  URL               https://github.com/lz4/lz4/archive/v1.7.5.tar.gz
  URL_HASH          SHA1=a710a7d45beb0951806d2b98f0c1739107e97c14
  CONFIGURE_COMMAND ""
  BUILD_IN_SOURCE 1
  BUILD_COMMAND     ${MAKE_EXE}
  UPDATE_COMMAND    ""
  INSTALL_COMMAND   ""
  PATCH_COMMAND			""
)

SET(LZ4_LIB_DIR ${CMAKE_CURRENT_BINARY_DIR}/deps/src/lz4_ext/lib)
SET(LZ4_INCLUDE_DIR ${CMAKE_CURRENT_BINARY_DIR}/deps/src/lz4_ext/include)
add_library(lz4 STATIC IMPORTED)
set_target_properties(lz4 PROPERTIES IMPORTED_LOCATION ${LZ4_LIB_DIR}/liblz4.a)

ExternalProject_Add(mariadb-client-library_ext
	PREFIX						deps
  GIT_REPOSITORY 		https://github.com/mariadb-corporation/mariadb-connector-c.git
	GIT_TAG						v3.1.4
  BUILD_IN_SOURCE 1
  CMAKE_ARGS        -DWITH_UNIT_TESTS="OFF" -DCMAKE_C_FLAGS_RELWITHDEBINFO="-ggdb" -DCMAKE_CXX_FLAGS_RELWITHDEBINFO="-ggdb"
                    -DCMAKE_BUILD_TYPE=RelWithDebInfo -DOPENSSL_ROOT_DIR=${OPENSSL_DIR} -DOPENSSL_LIBRARIES=${OPENSSL_LIB_DIR}
                    -DCLIENT_PLUGIN_MYSQL_CLEAR_PASSWORD=STATIC -DCLIENT_PLUGIN_SHA256_PASSWORD=STATIC -DCLIENT_PLUGIN_CACHING_SHA2_PASSWORD=STATIC
  PATCH_COMMAND			patch -p1 < ${PATCHES_DIR}/mariadb-connector-c-unified.patch
  UPDATE_COMMAND    ""
  INSTALL_COMMAND   ""
)

SET(MARIADB_CLIENT_LIB_DIR ${CMAKE_CURRENT_BINARY_DIR}/deps/src/mariadb-client-library_ext/libmariadb)
SET(MARIADB_CLIENT_INCLUDE_DIR ${CMAKE_CURRENT_BINARY_DIR}/deps/src/mariadb-client-library_ext/include)
add_library(mariadbclient STATIC IMPORTED)
set_target_properties(mariadbclient PROPERTIES IMPORTED_LOCATION ${MARIADB_CLIENT_LIB_DIR}/libmariadbclient.a)

ExternalProject_Add(pcre_ext
	PREFIX						deps
  URL               https://ftp.pcre.org/pub/pcre/pcre-8.39.tar.gz
  URL_HASH          SHA1=b3aec1f643d9701841e2f9d57ac121a7ff448fc8
  CONFIGURE_COMMAND ./configure
  BUILD_IN_SOURCE 1
  BUILD_COMMAND     ${MAKE_EXE}
  UPDATE_COMMAND    ""
  INSTALL_COMMAND   ""
  PATCH_COMMAND			""
)

SET(PCRE_LIB_DIR ${CMAKE_CURRENT_BINARY_DIR}/deps/src/pcre_ext/.libs)
SET(PCRE_INCLUDE_DIR ${CMAKE_CURRENT_BINARY_DIR}/deps/src/pcre_ext/include)
add_library(pcrecpp STATIC IMPORTED)
set_target_properties(pcrecpp PROPERTIES IMPORTED_LOCATION ${PCRE_LIB_DIR}/libpcrecpp.a)

ExternalProject_Add(sqlite3_ext
	PREFIX						deps
  URL               https://www.sqlite.org/2017/sqlite-amalgamation-3190200.zip
  URL_HASH          SHA1=ed35829ac78019528556809f41112b28a5d31e70
  CONFIGURE_COMMAND ""
  BUILD_IN_SOURCE 1
  BUILD_COMMAND     ${CMAKE_C_COMPILER} ${CMAKE_CFLAGS} -O2 -c -o sqlite3.o sqlite3.c -DSQLITE_ENABLE_MEMORY_MANAGEMENT -DSQLITE_ENABLE_JSON1
  UPDATE_COMMAND    ""
  INSTALL_COMMAND   ""
  PATCH_COMMAND			patch -p0 < ${CMAKE_CURRENT_SOURCE_DIR}/patches/sqlite3-from_unixtime.patch
)


SET(SQLITE3_DIR ${CMAKE_CURRENT_BINARY_DIR}/deps/src/sqlite3_ext)
SET(SQLITE3_LIB_DIR ${CMAKE_CURRENT_BINARY_DIR}/deps/src/sqlite3_ext/lib)
SET(SQLITE3_INCLUDE_DIR ${CMAKE_CURRENT_BINARY_DIR}/deps/src/sqlite3_ext/include)
add_library(sqlite3 STATIC IMPORTED)
set_target_properties(sqlite3 PROPERTIES IMPORTED_LOCATION ${SQLITE3_DIR}/sqlite3.o)

add_library(libproxysql STATIC lib/ProxySQL_GloVars.cpp lib/network.cpp lib/debug.cpp lib/configfile.cpp lib/Query_Cache.cpp
            lib/SpookyV2.cpp lib/MySQL_Authentication.cpp lib/gen_utils.cpp lib/sqlite3db.cpp lib/mysql_connection.cpp
            lib/MySQL_HostGroups_Manager.cpp lib/mysql_data_stream.cpp lib/MySQL_Thread.cpp lib/MySQL_Session.cpp lib/MySQL_Protocol.cpp
            lib/mysql_backend.cpp lib/Query_Processor.cpp lib/ProxySQL_Admin.cpp lib/ProxySQL_Config.cpp lib/ProxySQL_Restapi.cpp
            lib/MySQL_Monitor.cpp lib/MySQL_Logger.cpp lib/thread.cpp lib/MySQL_PreparedStatement.cpp lib/ProxySQL_Cluster.cpp
            lib/ClickHouse_Authentication.cpp lib/ClickHouse_Server.cpp lib/ProxySQL_Statistics.cpp lib/Chart_bundle_js.cpp
            lib/ProxySQL_HTTP_Server.cpp lib/ProxySQL_RESTAPI_Server.cpp lib/font-awesome.min.css.cpp lib/main-bundle.min.css.cpp
            lib/set_parser.cpp lib/c_tokenizer.c)

add_dependencies(libproxysql jemalloc_ext libev_ext cityhash_ext openssl_ext libmicrohttpd_ext libconfig_ext libhttpserver_ext libdaemon_ext curl_ext pcre_ext libre2_ext lz4_ext mariadb-client-library_ext libinjection_ext)

target_link_libraries(libproxysql PRIVATE nlohmann_json::nlohmann_json jemalloc libev cityhash libhttpserver ssl libconfig++ libdaemon curl pcrecpp libre2 lz4 mariadbclient libinjection microhttpd)

target_include_directories(libproxysql PRIVATE
                            "${CMAKE_CURRENT_SOURCE_DIR}/include"
                            "${MARIADB_CLIENT_INCLUDE_DIR}"
                            "${MARIADB_CLIENT_INCLUDE_DIR}"
                            "${OPENSSL_INCLUDE_DIR}"
                            "${CITIHASH_INCLUDE_DIR}"
                            "${LIBINJECTION_INCLUDE_DIR}"
                            "${JEMALLOC_INCLUDE_DIR}"
                            "${LIBCONFIG_INCLUDE_DIR}"
                            "${LIBDAEMON_INCLUDE_DIR}"
                            "${LIBHTTPSERVER_INCLUDE_DIR}"
                            "${LIBMICROHTTPD_INCLUDE_DIRS}"
													  "${LIBRE2_INCLUDE_DIR}"
													  "${PCRE_INCLUDE_DIR}"
                           )

# add the executable
add_executable(proxysql src/main.cpp src/proxysql_global.cpp src/SQLite3_Server.cpp)

target_include_directories(proxysql PRIVATE
                           "${CMAKE_CURRENT_SOURCE_DIR}/include"
                           "${MARIADB_CLIENT_INCLUDE_DIR}"
                           "${OPENSSL_INCLUDE_DIR}"
                           "${CITIHASH_INCLUDE_DIR}"
                           "${LIBINJECTION_INCLUDE_DIR}"
                           "${JEMALLOC_INCLUDE_DIR}"
                           "${LIBCONFIG_INCLUDE_DIR}"
                           "${LIBDAEMON_INCLUDE_DIR}"
                           "${LIBHTTPSERVER_INCLUDE_DIR}"
                           "${LIBMICROHTTPD_INCLUDE_DIRS}"
                           "${LIBRE2_INCLUDE_DIR}"
                           "${PCRE_INCLUDE_DIR}"
                           )

add_dependencies(proxysql openssl_ext jemalloc_ext libev_ext cityhash_ext libmicrohttpd_ext libhttpserver_ext libdaemon_ext libconfig_ext curl_ext pcre_ext libre2_ext lz4_ext mariadb-client-library_ext sqlite3_ext)

target_link_libraries(proxysql PUBLIC libproxysql nlohmann_json::nlohmann_json dl gnutls pthread curl ssl crypto sqlite3 pcrecpp pcre)