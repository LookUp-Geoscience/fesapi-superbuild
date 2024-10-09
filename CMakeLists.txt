cmake_minimum_required(VERSION 3.30)

project(fesapi-superbuild)

#### Here are located the FindPackages that we need
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake/Modules")
list(APPEND CMAKE_PREFIX_PATH "${CMAKE_CURRENT_BINARY_DIR}")

include(ExternalProject)
include(FetchContent)

if(WIN32)
  ExternalProject_Add(zlib

  URL http://www.zlib.net/zlib1211.zip
  URL_HASH SHA256=d7510a8ee1918b7d0cad197a089c0a2cd4d6df05fee22389f67f115e738b178d

  BUILD_ALWAYS OFF

  BINARY_DIR ${CMAKE_BINARY_DIR}/zlib-build
  INSTALL_DIR ${installDir}

  DOWNLOAD_EXTRACT_TIMESTAMP TRUE

  CMAKE_ARGS
    -DCMAKE_BUILD_TYPE:STRING=${BUILD_TYPE}
    -DCMAKE_INSTALL_PREFIX:PATH=<INSTALL_DIR>
    -DCMAKE_PREFIX_PATH:PATH=<INSTALL_DIR>
    -DBUILD_SHARED_LIBS:BOOL=ON    
  )
endif()



set(BOOST_DIR "${CMAKE_INSTALL_PREFIX}/boost")
set(BOOST_URL "${TPL_MIRROR_DIR}/boost_1_67_0.tar.gz")
message(STATUS "Extract Boost found at ${BOOST_URL}")

set(BOOST_ROOT ${CMAKE_CURRENT_BINARY_DIR}/boost/src/boost/ CACHE STRING "Set the value of BOOST_ROOT to point to the root folder of your boost install." FORCE)

ExternalProject_Add( boost
  URL ${BOOST_URL}
  PREFIX ${PROJECT_BINARY_DIR}/boost
  INSTALL_DIR ${BOOST_DIR}
  DOWNLOAD_EXTRACT_TIMESTAMP TRUE
  CONFIGURE_COMMAND ""
  BUILD_COMMAND ""
  INSTALL_COMMAND ""
  BUILD_IN_SOURCE 1
)

# Depending on the platform, the install directory could be 'lib' or 'lib64'.
# Fesapi doesn't use CMake Targets to import Minizip but needs an explicit
# path thus it has to be fixed regardless of the platform
set(MINIZIP_CMAKE_INSTALL_LIBDIR "lib" )

set(MINIZIP_DIR "${CMAKE_INSTALL_PREFIX}/minizip")
set(MINIZIP_URL "${TPL_MIRROR_DIR}/Minizip-1.1_cmake0.1.tar.gz")
message(STATUS "Building Minizip found at ${MINIZIP_URL}")

if( ${ENABLE_MPI} )
  set( MINIZIP_C_COMPILER ${MPI_C_COMPILER} )
  set( MINIZIP_CXX_COMPILER ${MPI_CXX_COMPILER} )
else()
  set( MINIZIP_C_COMPILER ${CMAKE_C_COMPILER} )
  set( MINIZIP_CXX_COMPILER ${CMAKE_CXX_COMPILER} )
endif()

ExternalProject_Add( minizip
  URL ${MINIZIP_URL}
  PREFIX ${PROJECT_BINARY_DIR}/minizip
  INSTALL_DIR ${MINIZIP_DIR}
  DOWNLOAD_EXTRACT_TIMESTAMP TRUE
  BUILD_COMMAND make -j ${NUM_PROC}
  INSTALL_COMMAND make install
  CMAKE_ARGS 
  	-DCMAKE_C_COMPILER=${MINIZIP_C_COMPILER}
  	-DCMAKE_CXX_COMPILER=${MINIZIP_CXX_COMPILER}
        -DCMAKE_INSTALL_PREFIX=<INSTALL_DIR>
        -DCMAKE_CXX_FLAGS=${TPL_CXX_STANDARD}
        -DCMAKE_PREFIX_PATH:PATH=<INSTALL_DIR>
        -DCMAKE_INSTALL_LIBDIR=${MINIZIP_CMAKE_INSTALL_LIBDIR}
)

if(WIN32)
  ExternalProject_Add_StepDependencies(minizip install zlib)
endif()

set( HDF5_DEPENDENCIES "" )
set(HDF5_DIR "${CMAKE_INSTALL_PREFIX}/hdf5")
set(HDF5_URL "${TPL_MIRROR_DIR}/hdf5-1.12.1.tar.gz")
message(STATUS "Building HDF5 found at ${HDF5_URL}")

if( ${ENABLE_MPI} )
  set( HDF5_C_COMPILER ${MPI_C_COMPILER} )
  set( HDF5_CXX_COMPILER ${MPI_CXX_COMPILER} )
  set( HDF5_ENABLE_PARALLEL "--enable-parallel")
else()
  set( HDF5_C_COMPILER ${CMAKE_C_COMPILER} )
  set( HDF5_CXX_COMPILER ${CMAKE_CXX_COMPILER} )
  set( HDF5_ENABLE_PARALLEL "")
endif()

set(HDF5_C_FLAGS "-fPIC ${C_FLAGS_NO_WARNINGS} ${CMAKE_C_FLAGS_RELEASE}")

ExternalProject_Add( hdf5
                     URL ${HDF5_URL}
                     PREFIX ${PROJECT_BINARY_DIR}/hdf5
                     INSTALL_DIR ${HDF5_DIR}
                     DOWNLOAD_EXTRACT_TIMESTAMP TRUE
                     BUILD_COMMAND make -j ${NUM_PROC}
                     INSTALL_COMMAND make install
                     CMAKE_ARGS -DCMAKE_C_COMPILER:STRING=${HDF5_C_COMPILER}
                                -DCMAKE_CXX_COMPILER:STRING=${HDF5_CXX_COMPILER}
                                -DCMAKE_C_FLAGS:STRING=${C_FLAGS_NO_WARNINGS}
                                -DCMAKE_C_FLAGS_RELEASE:STRING=${CMAKE_C_FLAGS_RELEASE}
                                -DCMAKE_CXX_FLAGS:STRING=${CXX_FLAGS_NO_WARNINGS}
                                -DCMAKE_CXX_FLAGS_RELEASE:STRING=${CMAKE_CXX_FLAGS_RELEASE}
                                -DCMAKE_BUILD_TYPE:STRING=${CMAKE_BUILD_TYPE}
                                -DCMAKE_VERBOSE_MAKEFILE:BOOL=${CMAKE_VERBOSE_MAKEFILE}
                                -DHDF5_ENABLE_PARALLEL:BOOL=${ENABLE_MPI}
                                -DCMAKE_POSITION_INDEPENDENT_CODE=${CMAKE_POSITION_INDEPENDENT_CODE}
                                -DCMAKE_INSTALL_PREFIX:PATH=<INSTALL_DIR>
                                -DBUILD_SHARED_LIBS:BOOL=ON
                                -DBUILD_STATIC_LIBS:BOOL=OFF
                                -DBUILD_TESTING:BOOL=OFF
                                -DHDF5_ENABLE_Z_LIB_SUPPORT:BOOL=ON
                                -DHDF5_BUILD_HL_LIB:BOOL=OFF
                                -DHDF5_BUILD_CPP_LIB:BOOL=OFF
                                -DHDF5_BUILD_EXAMPLES:BOOL=ON
                                -DHDF5_BUILD_TOOLS:BOOL=OFF
                                -DHDF5_BUILD_UTILS:BOOL=OFF
)

if( ENABLE_MPI )
  set( FESAPI_C_COMPILER ${MPI_C_COMPILER} )
  set( FESAPI_CXX_COMPILER ${MPI_CXX_COMPILER} )
else()
  set( FESAPI_C_COMPILER ${CMAKE_C_COMPILER} )
  set( FESAPI_CXX_COMPILER ${CMAKE_CXX_COMPILER} )
endif( ENABLE_MPI )

#
# Then Fesapi
#
set(FESAPI_DIR "${CMAKE_INSTALL_PREFIX}/fesapi")
set(FESAPI_URL "${TPL_MIRROR_DIR}/fesapi-2.10.1.0.tar.gz")

message(STATUS "Building Fesapi found at ${FESAPI_URL}")

#FetchContent_Declare(fesapi
	#GIT_REPOSITORY ${FESAPI_URL}
  #GIT_TAG ${FESAPI_TAG}
  #)
#FetchContent_GetProperties(fesapi)
#if(NOT fesapi_POPULATED)
	#FetchContent_Populate(fesapi)
  #endif()

ExternalProject_Add( fesapi
  URL ${FESAPI_URL}
  #SOURCE_DIR ${fesapi_SOURCE_DIR}  # Provided by FetchContent
  PREFIX ${PROJECT_BINARY_DIR}/fesapi
  INSTALL_DIR ${FESAPI_DIR}
  DOWNLOAD_EXTRACT_TIMESTAMP TRUE
  BUILD_COMMAND make -j ${NUM_PROC}
  INSTALL_COMMAND make install
  DEPENDS hdf5 minizip boost
  CMAKE_ARGS -DCMAKE_INSTALL_PREFIX=<INSTALL_DIR>
              -DCMAKE_C_COMPILER=${FESAPI_C_COMPILER}
              -DCMAKE_CXX_COMPILER=${FESAPI_CXX_COMPILER}
              -DCMAKE_INSTALL_RPATH_USE_LINK_PATH:BOOL=TRUE
              -DHDF5_ROOT:PATH=${HDF5_DIR}              
	            -DHDF5_PREFER_PARALLEL:BOOL=${ENABLE_MPI}
              -DMINIZIP_ROOT:PATH=${MINIZIP_DIR}
              -DBoost_NO_SYSTEM_PATHS:BOOL=TRUE
              -DBoost_NO_BOOST_CMAKE:BOOL=TRUE
              -DBoost_INCLUDE_DIR:PATH=${CMAKE_CURRENT_BINARY_DIR}/boost/src/boost              
              -DWITH_EXAMPLE:BOOL=OFF
              -DWITH_DOTNET_WRAPPING:BOOL=OFF
              -DWITH_JAVA_WRAPPING:BOOL=OFF
              -DWITH_PYTHON_WRAPPING:BOOL=OFF              
              -DWITH_TEST:BOOL=OFF
              -DWITH_RESQML2_2:BOOL=ON
)


ExternalProject_Add_StepDependencies(fesapi install minizip)
ExternalProject_Add_StepDependencies(fesapi install hdf5)






