﻿set( LIB_NAME_BASE xeveb )

set( LIB_SOVERSION 1)

file (GLOB ETM_INC "../inc/*.h")
file (GLOB LIB_API_SRC "xeve.c")
file (GLOB LIB_BASE_SRC "xeve_*.c")
file (GLOB LIB_BASE_INC "xeve_*.h" )
file (GLOB LIB_SSE_SRC "sse/xeve_*.c")
file (GLOB LIB_SSE_INC "sse/xeve_*.h" )
file (GLOB LIB_AVX_SRC "../src_base/avx/xeve_*.c")
file (GLOB LIB_AVX_INC "../src_base/avx/xeve_*.h" )
file (GLOB LIB_NEON_SRC "../src_base/neon/xeve_*.c")
file (GLOB LIB_NEON_INC "../src_base/neon/xeve_*.h" )


include(GenerateExportHeader)
include_directories("${CMAKE_BINARY_DIR}")




if("${ARM}" STREQUAL "TRUE")                      
  add_library( ${LIB_NAME_BASE} STATIC ${LIB_API_SRC} ${ETM_INC} ${LIB_BASE_SRC} ${LIB_BASE_INC} ${LIB_NEON_INC} ${LIB_NEON_SRC})
  add_library( ${LIB_NAME_BASE}_dynamic SHARED ${LIB_API_SRC} ${ETM_INC} ${LIB_BASE_SRC} ${LIB_BASE_INC} ${LIB_NEON_INC} ${LIB_NEON_SRC})
else()
  add_library( ${LIB_NAME_BASE} STATIC ${LIB_API_SRC} ${XEVE_INC} ${LIB_BASE_SRC} ${LIB_BASE_INC} ${LIB_SSE_SRC} ${LIB_SSE_INC}
                                     ${LIB_AVX_SRC} ${LIB_AVX_INC} )
  add_library( ${LIB_NAME_BASE}_dynamic SHARED ${LIB_API_SRC} ${XEVE_INC} ${LIB_BASE_SRC} ${LIB_BASE_INC} ${LIB_SSE_SRC} ${LIB_SSE_INC}
                                             ${LIB_AVX_SRC} ${LIB_AVX_INC} )
endif()


set_target_properties(${LIB_NAME_BASE}_dynamic PROPERTIES VERSION ${PROJECT_VERSION_MAJOR}.${PROJECT_VERSION_MINOR} SOVERSION ${LIB_SOVERSION})

# @todo Consider using WINDOWS_EXPORT_ALL_SYMBOLS instead of generate_export_header
# @see https://cmake.org/cmake/help/latest/prop_tgt/WINDOWS_EXPORT_ALL_SYMBOLS.html#prop_tgt:WINDOWS_EXPORT_ALL_SYMBOLS
#if(MSVC)
    # @see https://blog.kitware.com/create-dlls-on-windows-without-declspec-using-new-cmake-export-all-feature/
    # @see https://cmake.org/cmake/help/v3.3/module/GenerateExportHeader.html
    #
    #set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS TRUE)
    #set(BUILD_SHARED_LIBS TRUE)
#endif()

# Generate export macros for libraries
generate_export_header(${LIB_NAME_BASE}_dynamic           
                       BASE_NAME XEVE
                       EXPORT_FILE_NAME ${CMAKE_BINARY_DIR}/xeve_exports.h)

# This will cause the export macros to expand to nothing when building the static library.
set_target_properties(${LIB_NAME_BASE} PROPERTIES COMPILE_FLAGS -DLIBSHARED_AND_STATIC_STATIC_DEFINE)

source_group("base\\header" FILES ${LIB_BASE_INC} ${ETM_INC})
source_group("base\\source" FILES ${LIB_BASE_SRC} ${LIB_API_SRC})
source_group("base\\sse\\header" FILES ${LIB_SSE_INC})
source_group("base\\sse\\source" FILES ${LIB_SSE_SRC})
source_group("base\\avx\\header" FILES ${LIB_AVX_INC})
source_group("base\\avx\\source" FILES ${LIB_AVX_SRC})
source_group("base\\neon\\header" FILES ${LIB_NEON_INC})
source_group("base\\neon\\source" FILES ${LIB_NEON_SRC})

if("${ARM}" STREQUAL "TRUE")  
  include_directories( ${LIB_NAME_BASE} PUBLIC . .. ../inc ./neon)
else()
  include_directories( ${LIB_NAME_BASE} PUBLIC . .. ../inc ./sse ./avx)
endif()

set_target_properties(${LIB_NAME_BASE} PROPERTIES FOLDER lib
                                                  ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)

set( SSE ${BASE_INC_FILES} ${LIB_SSE_SRC})
set( AVX ${LIB_AVX_SRC} )
set( NEON ${LIB_NEON_SRC})

set_target_properties(${LIB_NAME_BASE}_dynamic PROPERTIES OUTPUT_NAME ${LIB_NAME_BASE})

if( MSVC )                      
  target_compile_definitions( ${LIB_NAME_BASE} PUBLIC ANY _CRT_SECURE_NO_WARNINGS )
  set_target_properties(${LIB_NAME_BASE}_dynamic PROPERTIES FOLDER lib
                                                            RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
elseif( UNIX OR MINGW )
  if("${ARM}" STREQUAL "FALSE")
    set_property( SOURCE ${SSE} APPEND PROPERTY COMPILE_FLAGS "-msse4.1" )
    set_property( SOURCE ${AVX} APPEND PROPERTY COMPILE_FLAGS " -mavx2" )
  endif()
  set_target_properties(${LIB_NAME_BASE}_dynamic PROPERTIES FOLDER lib
                                                            LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
  target_compile_definitions( ${LIB_NAME_BASE} PUBLIC ANY LINUX )  
  target_link_libraries(${LIB_NAME_BASE} m)
endif()

# Install rules
#
# Set DCMAKE_INSTALL_PREFIX to change default install prefix
# e.g cmake .. -DSET_PROF=BASE -DCMAKE_INSTALL_PREFIX='C:\Users\name\git\xeve_internal\build-windows\install'

# List the headers we want to declare as public for installation.
set(XEVE_PUBLIC_HEADERS "${ETM_INC}")
set_target_properties(${LIB_NAME_BASE} PROPERTIES PUBLIC_HEADER "${XEVE_PUBLIC_HEADERS}")

set(XEVE_PRIVATE_HEADERS "${LIB_BASE_INC}" "${LIB_SSE_INC}" "${LIB_AVX_INC}" "${LIB_NEON_INC}")

# Install static library and public headers
#
# Static library (libxeveb.a or xeveb.lib) will be installed to <prefix>/lib/xeveb
# Public headers will be installed to <prefix>/include/xeveb
#
include(GNUInstallDirs)
install(TARGETS ${LIB_NAME_BASE}
        ARCHIVE COMPONENT Development DESTINATION ${CMAKE_INSTALL_LIBDIR}/${LIB_NAME_BASE}
        PUBLIC_HEADER COMPONENT Development DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/${LIB_NAME_BASE}
)

# Install shared library
#
# On non-DLL platforms shared library (libxeveb.so) will be installed to <prefix>/lib/xeveb.
# On DLL platforms the shred DLL (xeveb.dll) will be installed to <prefix>/bin and its import library will be installed to <prefix>/lib/xeveb/import
#
install(TARGETS ${LIB_NAME_BASE}_dynamic
        RUNTIME COMPONENT Libraries DESTINATION ${CMAKE_INSTALL_BINDIR}
        LIBRARY 
            COMPONENT Libraries DESTINATION ${CMAKE_INSTALL_LIBDIR} 
            NAMELINK_COMPONENT Development DESTINATION ${CMAKE_INSTALL_LIBDIR} 
        ARCHIVE COMPONENT Development DESTINATION ${CMAKE_INSTALL_LIBDIR}/${LIB_NAME_BASE}/import
)

install( FILES
    ${PROJECT_BINARY_DIR}/xeve_exports.h DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/${LIB_NAME_BASE}
)

if( NOT MSVC )
    configure_file(
        "${CMAKE_SOURCE_DIR}/pkgconfig/${LIB_NAME_BASE}.pc.in"
        "${CMAKE_BINARY_DIR}/${LIB_NAME_BASE}.pc"
        IMMEDIATE @ONLY)

    install( FILES
        "${CMAKE_BINARY_DIR}/${LIB_NAME_BASE}.pc" DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig
    )
endif()
