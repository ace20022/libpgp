cmake_minimum_required(VERSION 3.0)

project(libpgp-error VERSION 1.2.7 LANGUAGES C)

#find_package(freetype 2.6 REQUIRED NO_MODULE)
#find_package(iconv 1 REQUIRED)
#find_package(libxml2 2.9 REQUIRED NO_MODULE)

if(MSVC)
  set(CMAKE_DEBUG_POSTFIX "d")
endif()

if (EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/cmake/awk.exe)
  set(AWK_CMD ${CMAKE_CURRENT_SOURCE_DIR}/cmake/awk)
  message("AWK command: ${AWK_CMD}")

else()
  message("AWK path not set, can't generate error codes")
  return()
endif()

set(SRCS
  ${CMAKE_CURRENT_SOURCE_DIR}/cmake/libgpg-error.def
  ${CMAKE_CURRENT_SOURCE_DIR}/src/b64dec.c
  ${CMAKE_CURRENT_SOURCE_DIR}/src/code-from-errno.c
  ${CMAKE_CURRENT_SOURCE_DIR}/src/code-to-errno.c
  ${CMAKE_CURRENT_SOURCE_DIR}/src/err-codes.h
  ${CMAKE_CURRENT_SOURCE_DIR}/src/err-sources.h
  ${CMAKE_CURRENT_SOURCE_DIR}/src/estream-printf.c
  ${CMAKE_CURRENT_SOURCE_DIR}/src/estream-printf.h
  ${CMAKE_CURRENT_SOURCE_DIR}/src/estream.c
  ${CMAKE_CURRENT_SOURCE_DIR}/src/gettext.h
  ${CMAKE_CURRENT_SOURCE_DIR}/src/gpg-error.c
  ${CMAKE_CURRENT_SOURCE_DIR}/src/gpgrt-int.h
  ${CMAKE_CURRENT_SOURCE_DIR}/src/init.c
  ${CMAKE_CURRENT_SOURCE_DIR}/src/init.h
  ${CMAKE_CURRENT_SOURCE_DIR}/src/lock.h
  ${CMAKE_CURRENT_SOURCE_DIR}/src/strerror-sym.c
  ${CMAKE_CURRENT_SOURCE_DIR}/src/strerror.c
  ${CMAKE_CURRENT_SOURCE_DIR}/src/strsource-sym.c
  ${CMAKE_CURRENT_SOURCE_DIR}/src/strsource.c
  ${CMAKE_CURRENT_SOURCE_DIR}/src/syscfg/lock-obj-pub.mingw32.h
  ${CMAKE_CURRENT_SOURCE_DIR}/src/thread.h
  ${CMAKE_CURRENT_SOURCE_DIR}/src/version.c
  ${CMAKE_CURRENT_SOURCE_DIR}/src/visibility.c
  ${CMAKE_CURRENT_SOURCE_DIR}/src/visibility.h
  ${CMAKE_CURRENT_SOURCE_DIR}/src/w32-add.h
  ${CMAKE_CURRENT_SOURCE_DIR}/src/w32-estream.c
  ${CMAKE_CURRENT_SOURCE_DIR}/src/w32-gettext.c
  ${CMAKE_CURRENT_SOURCE_DIR}/src/w32-iconv.c
  ${CMAKE_CURRENT_SOURCE_DIR}/src/w32-lock-obj.h
  ${CMAKE_CURRENT_SOURCE_DIR}/src/w32-lock.c
  ${CMAKE_CURRENT_SOURCE_DIR}/src/w32-thread.c
)

ADD_LIBRARY(libpgp-error SHARED ${SRCS})
SET_TARGET_PROPERTIES(libpgp-error PROPERTIES LINK_FLAGS "/DEF:\"libgpg-error.def\"")

target_compile_definitions(libpgp-error
  PRIVATE
  HAVE_CONFIG_H
  PKGDATADIR=""
  _CRT_SECURE_NO_WARNINGS
  _CRT_NONSTDC_NO_WARNINGS)

target_include_directories(libpgp-error
  PRIVATE
  $<BUILD_INTERFACE:.;cmake;src;src/syscfg;${CMAKE_CURRENT_BINARY_DIR}>
  INTERFACE
  $<INSTALL_INTERFACE:include/libpgp-errors>
)

add_custom_command(
   TARGET libpgp-error
   PRE_BUILD
   COMMAND ${AWK_CMD} -f ${CMAKE_CURRENT_SOURCE_DIR}/src/mkstrtable.awk -v textidx=3 ${CMAKE_CURRENT_SOURCE_DIR}/src/err-sources.h.in > ${CMAKE_CURRENT_SOURCE_DIR}/cmake/err-sources.h
   COMMAND ${AWK_CMD} -f ${CMAKE_CURRENT_SOURCE_DIR}/src/mkstrtable.awk -v textidx=2 -v nogettext=1 ${CMAKE_CURRENT_SOURCE_DIR}/src/err-sources.h.in > ${CMAKE_CURRENT_SOURCE_DIR}/cmake/err-sources-sym.h
   COMMAND ${AWK_CMD} -f ${CMAKE_CURRENT_SOURCE_DIR}/src/mkstrtable.awk -v textidx=3 ${CMAKE_CURRENT_SOURCE_DIR}/src/err-codes.h.in > ${CMAKE_CURRENT_SOURCE_DIR}/cmake/err-codes.h
   COMMAND ${AWK_CMD} -f ${CMAKE_CURRENT_SOURCE_DIR}/src/mkstrtable.awk -v textidx=2 -v nogettext=1 ${CMAKE_CURRENT_SOURCE_DIR}/src/err-codes.h.in > ${CMAKE_CURRENT_SOURCE_DIR}/cmake/err-codes-sym.h
   COMMAND ${AWK_CMD} -f ${CMAKE_CURRENT_SOURCE_DIR}/src/mkerrnos.awk ${CMAKE_CURRENT_SOURCE_DIR}/src/errnos.in > ${CMAKE_CURRENT_SOURCE_DIR}/cmake/code-to-errno.h
   COMMAND ${AWK_CMD} -f ${CMAKE_CURRENT_SOURCE_DIR}/src/mkstrtable.awk -v textidx=2 -v nogettext=1 -v prefix=GPG_ERR_ -v namespace=errnos_ ${CMAKE_CURRENT_SOURCE_DIR}/src/errnos.in > ${CMAKE_CURRENT_SOURCE_DIR}/cmake/errnos-sym.h
)

target_link_libraries(libpgp-error PRIVATE Ws2_32.lib)

set(MKH_SRC
  ${CMAKE_CURRENT_SOURCE_DIR}/src/mkheader.c
)

ADD_EXECUTABLE(mkheader ${MKH_SRC})
target_compile_definitions(mkheader
  PRIVATE
  HAVE_CONFIG_H
  _CRT_SECURE_NO_WARNINGS
  _CRT_NONSTDC_NO_WARNINGS)
  
target_include_directories(mkheader
  PRIVATE
  $<BUILD_INTERFACE:.;cmake;src;${CMAKE_CURRENT_BINARY_DIR}>
  INTERFACE
  $<INSTALL_INTERFACE:include/libpgp-errors>
)
  
add_custom_command(
   TARGET mkheader
   POST_BUILD
   COMMAND ${CMAKE_COMMAND} -E chdir ${CMAKE_CURRENT_SOURCE_DIR}/src $<TARGET_FILE:mkheader> "mingw32" "win32-msvc" "gpg-error.h.in" ${CMAKE_CURRENT_SOURCE_DIR}/cmake/config.h "1.27" "0x011b00" > ${CMAKE_CURRENT_SOURCE_DIR}/cmake/gpg-error.h
)

set(MKEC_SRC
  ${CMAKE_CURRENT_SOURCE_DIR}/src/mkerrcodes.c
)

ADD_EXECUTABLE(mkerrcodes ${MKEC_SRC})
target_compile_definitions(mkheader
  PRIVATE
  HAVE_CONFIG_H
  _CRT_SECURE_NO_WARNINGS
  _CRT_NONSTDC_NO_WARNINGS)
  
target_include_directories(mkerrcodes
  PRIVATE
  $<BUILD_INTERFACE:.;cmake;src;${CMAKE_CURRENT_BINARY_DIR}>
  INTERFACE
  $<INSTALL_INTERFACE:include/libpgp-errors>
)

add_custom_command(
   TARGET mkerrcodes
   PRE_BUILD 
   COMMAND ${AWK_CMD} -f ${CMAKE_CURRENT_SOURCE_DIR}/src/mkerrcodes1.awk ${CMAKE_CURRENT_SOURCE_DIR}/src/errnos.in > ${CMAKE_CURRENT_SOURCE_DIR}/cmake/_mkerrcodes.h
   COMMAND cl -E ${CMAKE_CURRENT_SOURCE_DIR}/cmake/_mkerrcodes.h | findstr GPG_ERR_ | \"${AWK_CMD}\" -f ${CMAKE_CURRENT_SOURCE_DIR}/src/mkerrcodes.awk > ${CMAKE_CURRENT_SOURCE_DIR}/cmake/mkerrcodes.h
   COMMAND ${CMAKE_COMMAND} -E remove ${CMAKE_CURRENT_SOURCE_DIR}/cmake/_mkerrcodes.h
)

add_custom_command(
   TARGET mkerrcodes
   POST_BUILD 
   COMMAND $<TARGET_FILE:mkerrcodes> | \"${AWK_CMD}\" -f ${CMAKE_CURRENT_SOURCE_DIR}/src/mkerrcodes2.awk > ${CMAKE_CURRENT_SOURCE_DIR}/cmake/code-from-errno.h
)

add_dependencies(libpgp-error mkheader)
add_dependencies(libpgp-error mkerrcodes)
