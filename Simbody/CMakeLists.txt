#---------------------------------------------------
# Simbody 
#
# Creates SimTK Core library, base name=SimTKsimbody.
# Default libraries are shared & optimized. Variants
# are created for static (_static) and debug (_d) and
# provision is made for an optional "namespace" (ns)
# and version number (vn).
#
# Windows:
#   [ns_]SimTKsimbody[_vn][_d].dll
#   [ns_]SimTKsimbody[_vn][_d].lib
#   [ns_]SimTKsimbody[_vn]_static[_d].lib
# Unix:
#   lib[ns_]SimTKsimbody[_vn][_d].so
#   lib[ns_]SimTKsimbody[_vn]_static[_d].a
#
# All libraries are installed in 
#   %ProgramFiles%\SimTK\lib[64]  (Windows)
#   /usr/local/SimTK/lib[64]        (UNIX)
#
#----------------------------------------------------

cmake_minimum_required(VERSION 2.8)

if(COMMAND cmake_policy)
        cmake_policy(SET CMP0003 NEW)
        cmake_policy(SET CMP0005 NEW)
endif(COMMAND cmake_policy)

ADD_SUBDIRECTORY( SimTKcommon )
ADD_SUBDIRECTORY( SimTKmath )

PROJECT (Simbody)

# Use this to generate a private set of libraries whose names
# won't conflict with installed versions.
SET(BUILD_USING_NAMESPACE "" CACHE STRING
	"All library names will be prefixed with 'xxx_' if this is set to xxx.")

SET(BUILD_UNVERSIONED_LIBRARIES TRUE CACHE BOOL
 "Build library names, and assume dependency names, with no version numbers?")

SET(BUILD_VERSIONED_LIBRARIES FALSE CACHE BOOL
 "Build library names, and assume dependency names, with version numbers?")

SET(NS)
IF(BUILD_USING_NAMESPACE)
    SET(NS "${BUILD_USING_NAMESPACE}_")
ENDIF(BUILD_USING_NAMESPACE)


# Caution: this variable is automatically created by the CMake
# ENABLE_TESTING() command, but we'll take it over here for
# our own purposes too.
SET( BUILD_TESTING ON CACHE BOOL
	"Control building of Simbody test programs." )

SET( BUILD_EXAMPLES ON CACHE BOOL
	"Control building of Simbody example programs.")

# Set whether to build the Visualizer code.
SET(BUILD_VISUALIZER ON CACHE BOOL 
	"Control building of VisualizerGUI component")

# Turning this off reduces the build time (and space) substantially,
# but you may miss the occasional odd bug. Also currently on Windows it
# is easier to debug the static tests than the DLL-liked ones.
SET( BUILD_TESTING_STATIC ON CACHE BOOL
	"If building test or example programs, include statically-linked ones?" )

SET( BUILD_TESTING_SHARED ON CACHE BOOL
 	"If building test or example programs, include dynamically-linked ones?" )


# The source is organized into subdirectories, but we handle them all from
# this CMakeLists file rather than letting CMake visit them as SUBDIRS.
SET(SIMBODY_SOURCE_SUBDIRS .)
IF (BUILD_VISUALIZER)
    SET(SIMBODY_SOURCE_SUBDIRS ${SIMBODY_SOURCE_SUBDIRS} Visualizer)
ENDIF (BUILD_VISUALIZER)

# 
# Make sure "sandbox" input & output directories are set. During
# SimTK Core build, the build system will set them. Otherwise, we'll
# set them to sensible local values.
# If SimTK_INSTALL_PREFIX is set then
# it is a sandbox installation area, otherwise we want to install
# in /usr/local/SimTK or %ProgramFiles%\SimTK. Similarly, SimTK_SDK
# tells us where to find other SimTK Core modules on which this one
# depends. If it is not set, we'll look in /usr/local/SimTK or
# %ProgramFiles%\SimTK *regardless* of SimTK_INSTALL_PREFIX.
#

IF(${CMAKE_C_COMPILER} MATCHES "gcc")

    IF(NOT SimTK_INSTALL_PREFIX)
        SET(SimTK_INSTALL_PREFIX "/usr/local/SimTK" 
            CACHE PATH "Install directory")
    ENDIF(NOT SimTK_INSTALL_PREFIX)

    IF(NOT SimTK_SDK)
        SET(SimTK_SDK "/usr/local/SimTK" 
            CACHE PATH "SimTK dependency path" )
    ENDIF(NOT SimTK_SDK)

ELSE(${CMAKE_C_COMPILER} MATCHES "gcc")

    IF(NOT SimTK_INSTALL_PREFIX)
        SET(SimTK_INSTALL_PREFIX "$ENV{ProgramFiles}/SimTK" 
            CACHE PATH "Install directory")
    ENDIF(NOT SimTK_INSTALL_PREFIX)

    IF(NOT SimTK_SDK)
        SET(SimTK_SDK "$ENV{ProgramFiles}/SimTK" 
            CACHE PATH "SimTK dependency path")
    ENDIF(NOT SimTK_SDK)

ENDIF(${CMAKE_C_COMPILER} MATCHES "gcc")

mark_as_advanced(CMAKE_INSTALL_PREFIX)

INCLUDE_DIRECTORIES(${SimTK_SDK}/include)

IF( ${CMAKE_SIZEOF_VOID_P} EQUAL 8 )
  SET( LIB64 64 )
ELSE( ${CMAKE_SIZEOF_VOID_P} EQUAL 8 )
  SET( LIB64  )
ENDIF( ${CMAKE_SIZEOF_VOID_P} EQUAL 8 )

# Allow build of binaries on Leopard that work on Tiger
# Plus, -m32 is required for building on Snow Leopard, until we have a 64-bit Mac Lapack
IF( APPLE )
   SET( CMAKE_CXX_FLAGS "-mmacosx-version-min=10.4 -m32" )
   SET( CMAKE_C_FLAGS "-mmacosx-version-min=10.4 -m32" )
   SET( LIB64  )
ENDIF( APPLE )

IF(UNIX AND NOT CMAKE_BUILD_TYPE)
    SET(CMAKE_BUILD_TYPE Debug CACHE STRING "Debug or Release build" FORCE)
ENDIF (UNIX AND NOT CMAKE_BUILD_TYPE)

## Choose the maximum level of x86 instruction set that the compiler is 
## allowed to use. SSE2 is ubiquitous enough now that we don't mind
## abandoning machines that can't handle those instructions. SSE3 migh
## also be reasonable by now (April 2009) so this default should be
## revisited soon. This can be set to a different value by the person
## running CMake.
SET(BUILD_INST_SET "sse2"  # use SSE2 instruction set by default
    CACHE STRING "CPU instruction level compiler is permitted to use.")
MARK_AS_ADVANCED( BUILD_INST_SET )

## When building in any of the Release modes, tell gcc to use full optimization and
## to generate SSE2 floating point instructions. Here we are specifying *all* of the
## Release flags, overriding CMake's defaults.
## Watch out for optimizer bugs in particular gcc versions!

IF(${CMAKE_C_COMPILER} MATCHES "gcc")
    string(TOLOWER ${BUILD_INST_SET} GCC_INST_SET)

    # Get the gcc version number in major.minor.build format
    execute_process(COMMAND ${CMAKE_C_COMPILER} -dumpversion
                    OUTPUT_VARIABLE GCC_VERSION)

    # Unrolling fixed-count loops is a useful optimization for Simmatrix.
    SET(GCC_OPT_ENABLE "-funroll-loops")

    # If you know of optimization bugs that affect SimTK in particular
    # gcc versions, this is the place to turn off those optimizations.
    SET(GCC_OPT_DISABLE)
    # We know Gcc 4.4.3 on Ubuntu 10 is buggy and that Snow Leopard's
    # 4.2.1 is not. To be safe for now we'll assume anything over 4.3
    # should have these disabled.
    if (GCC_VERSION VERSION_GREATER 4.3 OR GCC_VERSION VERSION_EQUAL 4.3)
        SET(GCC_OPT_DISABLE 
	"-fno-tree-vrp -fno-strict-aliasing -fno-guess-branch-probability")
    endif()

    # C++
    SET(CMAKE_CXX_FLAGS_DEBUG          "-g -m${GCC_INST_SET}" 
      CACHE STRING "g++ Debug build compile flags" FORCE)
    SET(CMAKE_CXX_FLAGS_RELEASE        
      "-DNDEBUG -O3 ${GCC_OPT_ENABLE} ${GCC_OPT_DISABLE} -m${GCC_INST_SET}" 
      CACHE STRING "g++ Release build compile flags" FORCE)
    SET(CMAKE_CXX_FLAGS_RELWITHDEBINFO 
      "-DNDEBUG -O3 -g ${GCC_OPT_ENABLE} ${GCC_OPT_DISABLE} -m${GCC_INST_SET}" 
      CACHE STRING "g++ RelWithDebInfo build compile flags" FORCE)
    SET(CMAKE_CXX_FLAGS_MINSIZEREL     "-DNDEBUG -Os -m${GCC_INST_SET}" 
      CACHE STRING "g++ MinSizeRel build compile flags" FORCE)

    # C
    SET(CMAKE_C_FLAGS_DEBUG            "-g -m${GCC_INST_SET}" 
      CACHE STRING "gcc Debug build compile flags" FORCE)
    SET(CMAKE_C_FLAGS_RELEASE          
      "-DNDEBUG -O3 ${GCC_OPT_ENABLE} ${GCC_OPT_DISABLE} -m${GCC_INST_SET}" 
      CACHE STRING "gcc Release build compile flags" FORCE)
    SET(CMAKE_C_FLAGS_RELWITHDEBINFO   
      "-DNDEBUG -O3 -g ${GCC_OPT_ENABLE} ${GCC_OPT_DISABLE} -m${GCC_INST_SET}" 
      CACHE STRING "gcc RelWithDebInfo build compile flags" FORCE)
    SET(CMAKE_C_FLAGS_MINSIZEREL       "-DNDEBUG -Os -m${GCC_INST_SET}" 
      CACHE STRING "gcc MinSizeRel build compile flags" FORCE)

ENDIF(${CMAKE_C_COMPILER} MATCHES "gcc")

## When building in any of the Release modes, tell VC++ cl compiler to use intrinsics 
## (i.e. sqrt instruction rather than sqrt subroutine) with flag /Oi.

IF(${CMAKE_C_COMPILER} MATCHES "cl")
    STRING(TOUPPER ${BUILD_INST_SET} CL_INST_SET)

    ## C++
    SET(CMAKE_CXX_FLAGS_DEBUG        
	"/D _DEBUG /MDd /Od /Ob0 /RTC1 /Zi /GS- /arch:${CL_INST_SET}" 
        CACHE STRING "VC++ Debug build compile flags" FORCE)
    SET(CMAKE_CXX_FLAGS_RELEASE        
	"/D NDEBUG /MD  /O2 /Ob2 /Oi /GS- /arch:${CL_INST_SET}" 
        CACHE STRING "VC++ Release build compile flags" FORCE)
    SET(CMAKE_CXX_FLAGS_RELWITHDEBINFO 
	"/D NDEBUG /MD  /O2 /Ob2 /Oi /Zi /GS- /arch:${CL_INST_SET}" 
        CACHE STRING "VC++ RelWithDebInfo build compile flags" FORCE)
    SET(CMAKE_CXX_FLAGS_MINSIZEREL 
	"/D NDEBUG /MD  /O1 /Ob1 /Oi /GS- /arch:${CL_INST_SET}" 
        CACHE STRING "VC++ MinSizeRel build compile flags" FORCE)

    ## C
    SET(CMAKE_C_FLAGS_DEBUG        
	"/D _DEBUG /MDd /Od /Ob0 /RTC1 /Zi /GS- /arch:${CL_INST_SET}" 
        CACHE STRING "VC++ Debug build compile flags" FORCE)
    SET(CMAKE_C_FLAGS_RELEASE        
	"/D NDEBUG /MD  /O2 /Ob2 /Oi /GS- /arch:${CL_INST_SET}" 
        CACHE STRING "VC++ Release build compile flags" FORCE)
    SET(CMAKE_C_FLAGS_RELWITHDEBINFO 
	"/D NDEBUG /MD  /O2 /Ob2 /Oi /Zi /GS- /arch:${CL_INST_SET}" 
        CACHE STRING "VC++ RelWithDebInfo build compile flags" FORCE)
    SET(CMAKE_C_FLAGS_MINSIZEREL 
	"/D NDEBUG /MD  /O1 /Ob1 /Oi /GS- /arch:${CL_INST_SET}" 
        CACHE STRING "VC++ MinSizeRel build compile flags" FORCE)

ENDIF(${CMAKE_C_COMPILER} MATCHES "cl")

SET(SIMBODY_LIBRARY_NAME ${NS}SimTKsimbody CACHE STRING
"Base name of the library being built; can't be changed here; see BUILD_USING_NAMESPACE variable."
FORCE)

# Collect up information about the version of the simbody library we're building
# and make it available to the code so it can be built into the binaries.

SET(SIMBODY_MAJOR_VERSION 2)
SET(SIMBODY_MINOR_VERSION 2)
SET(SIMBODY_BUILD_VERSION 0)

# Report the version number to the CMake UI. Don't include the 
# build version if it is zero.
SET(BUILD_VERSION_STRING)
IF(SIMBODY_BUILD_VERSION)
    SET(BUILD_VERSION_STRING ".${SIMBODY_BUILD_VERSION}")
ENDIF(SIMBODY_BUILD_VERSION)

SET(SIMBODY_VERSION 
    "${SIMBODY_MAJOR_VERSION}.${SIMBODY_MINOR_VERSION}${BUILD_VERSION_STRING}" CACHE STRING 
	 "This is the version that will be built (can't be changed here)." FORCE)

# This is the suffix if we're building and depending on versioned libraries.
SET(VN "_${SIMBODY_VERSION}")

SET(SIMBODY_COPYRIGHT_YEARS "2005-9")

# underbar separated list of dotted authors, no spaces or commas
SET(SIMBODY_AUTHORS "Michael.Sherman_Peter.Eastman")

# Get the subversion revision number if we can
# It's possible that WIN32 installs use svnversion through cygwin
# so we'll try for both svnversion.exe and svnversion. Note that
# this will result in warnings if all you have is Tortoise without
# Cygwin, and your "about" string will say "unknown" rather than
# providing the SVN version of the source.

FIND_PROGRAM (SVNVERSION_EXE svnversion.exe)
IF (SVNVERSION_EXE)
    EXEC_PROGRAM (${SVNVERSION_EXE}
                  ARGS  \"${PROJECT_SOURCE_DIR}\"
                  OUTPUT_VARIABLE SIMBODY_SVN_REVISION )
ELSE (SVNVERSION_EXE) 
    FIND_PROGRAM (SVNVERSION svnversion) 
    IF (SVNVERSION)
        EXEC_PROGRAM (${SVNVERSION}
                      ARGS "${PROJECT_SOURCE_DIR}"
                      OUTPUT_VARIABLE SIMBODY_SVN_REVISION)
    ELSE (SVNVERSION)
        MESSAGE (STATUS 
          "Could not find 'svnversion' executable; 'about' will be wrong. (Cygwin provides one on Windows.)"
        )
        SET (SIMBODY_SVN_REVISION unknown) 
    ENDIF (SVNVERSION)
ENDIF (SVNVERSION_EXE)
# Remove colon from build version, for easier placement in directory names
STRING(REPLACE ":" "_" SIMBODY_SVN_REVISION ${SIMBODY_SVN_REVISION})

ADD_DEFINITIONS(-DSimTK_SIMBODY_LIBRARY_NAME=${SIMBODY_LIBRARY_NAME}
                -DSimTK_SIMBODY_MAJOR_VERSION=${SIMBODY_MAJOR_VERSION}
                -DSimTK_SIMBODY_MINOR_VERSION=${SIMBODY_MINOR_VERSION}
		-DSimTK_SIMBODY_BUILD_VERSION=${SIMBODY_BUILD_VERSION})

# CMake quotes automatically when building Visual Studio projects but we need
# to add them ourselves for Linux or Cygwin. Two cases to avoid duplicate quotes
# in Visual Studio which end up in the binary.

IF (${CMAKE_GENERATOR} MATCHES "Visual Studio")
   SET(NEED_QUOTES FALSE)
ELSE (${CMAKE_GENERATOR} MATCHES "Visual Studio")
   SET(NEED_QUOTES TRUE)
ENDIF (${CMAKE_GENERATOR} MATCHES "Visual Studio")

##TODO: doesn't work without quotes in nightly build
SET(NEED_QUOTES TRUE)

IF(NEED_QUOTES)
   ADD_DEFINITIONS(-DSimTK_SIMBODY_SVN_REVISION="${SIMBODY_SVN_REVISION}"
                   -DSimTK_SIMBODY_COPYRIGHT_YEARS="${SIMBODY_COPYRIGHT_YEARS}"
                   -DSimTK_SIMBODY_AUTHORS="${SIMBODY_AUTHORS}")
ELSE(NEED_QUOTES)
   ADD_DEFINITIONS(-DSimTK_SIMBODY_SVN_REVISION=${SIMBODY_SVN_REVISION}
                   -DSimTK_SIMBODY_COPYRIGHT_YEARS=${SIMBODY_COPYRIGHT_YEARS}
                   -DSimTK_SIMBODY_AUTHORS=${SIMBODY_AUTHORS})
ENDIF(NEED_QUOTES)

# -DSimTK_SIMBODY_TYPE has to be defined in the target subdirectories.
# -Dsimbody_EXPORTS defined automatically when Windows DLL build is being done.


SET(SHARED_TARGET ${SIMBODY_LIBRARY_NAME})
SET(STATIC_TARGET ${SIMBODY_LIBRARY_NAME}_static)
SET(SHARED_TARGET_VN ${SIMBODY_LIBRARY_NAME}${VN})
SET(STATIC_TARGET_VN ${SIMBODY_LIBRARY_NAME}${VN}_static)

# Ensure that debug libraries have "_d" appended to their names.
# CMake gets this right on Windows automatically with this definition.
IF (${CMAKE_GENERATOR} MATCHES "Visual Studio")
    SET(CMAKE_DEBUG_POSTFIX "_d" CACHE INTERNAL "" FORCE)
ENDIF (${CMAKE_GENERATOR} MATCHES "Visual Studio")

# But on Unix or Cygwin we have to add the suffix manually
IF (UNIX AND CMAKE_BUILD_TYPE MATCHES Debug)
    SET(SHARED_TARGET ${SHARED_TARGET}_d)
    SET(STATIC_TARGET ${STATIC_TARGET}_d)
    SET(SHARED_TARGET_VN ${SHARED_TARGET_VN}_d)
    SET(STATIC_TARGET_VN ${STATIC_TARGET_VN}_d)
ENDIF (UNIX AND CMAKE_BUILD_TYPE MATCHES Debug)

## Test against the unversioned libraries if they are being built;
## otherwise against the versioned libraries.
IF(BUILD_UNVERSIONED_LIBRARIES)
	SET(TEST_SHARED_TARGET ${SHARED_TARGET})
	SET(TEST_STATIC_TARGET ${STATIC_TARGET})
ELSE(BUILD_UNVERSIONED_LIBRARIES)
	SET(TEST_SHARED_TARGET ${SHARED_TARGET_VN})
	SET(TEST_STATIC_TARGET ${STATIC_TARGET_VN})
ENDIF(BUILD_UNVERSIONED_LIBRARIES)


## If no one says otherwise, change the executable path to drop into the same binary
## location as the DLLs so that the test cases will use the just-build DLLs.
IF(NOT EXECUTABLE_OUTPUT_PATH)
  SET(EXECUTABLE_OUTPUT_PATH ${PROJECT_BINARY_DIR} 
      CACHE INTERNAL "Single output directory for building all executables.")
ENDIF(NOT EXECUTABLE_OUTPUT_PATH)
IF(NOT LIBRARY_OUTPUT_PATH)
  SET(LIBRARY_OUTPUT_PATH ${PROJECT_BINARY_DIR} 
      CACHE INTERNAL "Single output directory for building all libraries.")
ENDIF(NOT LIBRARY_OUTPUT_PATH)
SET(${PROJECT_NAME}_EXECUTABLE_DIR ${EXECUTABLE_OUTPUT_PATH}/${CMAKE_CFG_INTDIR})
SET(${PROJECT_NAME}_LIBRARY_DIR    ${LIBRARY_OUTPUT_PATH}/${CMAKE_CFG_INTDIR})


SET(SimTKCOMMON_LIBRARY_NAME ${NS}SimTKcommon)
SET(SimTKCOMMON_SHARED_LIBRARY ${SimTKCOMMON_LIBRARY_NAME})
SET(SimTKCOMMON_STATIC_LIBRARY ${SimTKCOMMON_LIBRARY_NAME}_static)

SET(SimTKCOMMON_LIBRARY_NAME_VN ${NS}SimTKcommon${VN})
SET(SimTKCOMMON_SHARED_LIBRARY_VN ${SimTKCOMMON_LIBRARY_NAME_VN})
SET(SimTKCOMMON_STATIC_LIBRARY_VN ${SimTKCOMMON_LIBRARY_NAME_VN}_static)

SET(SimTKMATH_LIBRARY_NAME ${NS}SimTKmath)
SET(SimTKMATH_SHARED_LIBRARY ${SimTKMATH_LIBRARY_NAME})
SET(SimTKMATH_STATIC_LIBRARY ${SimTKMATH_LIBRARY_NAME}_static)

SET(SimTKMATH_LIBRARY_NAME_VN ${NS}SimTKmath${VN})
SET(SimTKMATH_SHARED_LIBRARY_VN ${SimTKMATH_LIBRARY_NAME_VN})
SET(SimTKMATH_STATIC_LIBRARY_VN ${SimTKMATH_LIBRARY_NAME_VN}_static)

# Determine which math libraries to use for this platform.
SET(BUILD_USING_OTHER_LAPACK "" CACHE STRING
 "If you have your own Lapack, put its library basename here (namespace and version don't apply). Default is to use SimTKlapack, with namespace and version if any.")

SET(LAPACK_BEING_USED ${NS}SimTKlapack CACHE STRING
"Basename of the actual Lapack library we're depending on; can't change here; see variable BUILD_USING_OTHER_LAPACK." FORCE)
SET(LAPACK_BEING_USED_VN ${NS}SimTKlapack${VN})

## Note that the version number does not apply when the user has
## specified an alternate Lapack library.
IF(BUILD_USING_OTHER_LAPACK)
    SET(LAPACK_BEING_USED ${BUILD_USING_OTHER_LAPACK} CACHE STRING
"Basename of the actual Lapack library we're depending on; can't change here; see variable BUILD_USING_OTHER_LAPACK." FORCE)
    SET(LAPACK_BEING_USED_VN ${BUILD_USING_OTHER_LAPACK})
ENDIF(BUILD_USING_OTHER_LAPACK)

IF(${CMAKE_C_COMPILER} MATCHES "gcc")
    SET(MATH_LIBS_TO_USE    ${LAPACK_BEING_USED} pthread dl)
    SET(MATH_LIBS_TO_USE_VN ${LAPACK_BEING_USED_VN} pthread dl)
ELSE(${CMAKE_C_COMPILER} MATCHES "gcc")
    ## Assume Microsoft Visual Studio
    SET(MATH_LIBS_TO_USE    ${LAPACK_BEING_USED} pthreadVC2)
    SET(MATH_LIBS_TO_USE_VN ${LAPACK_BEING_USED_VN} pthreadVC2)
ENDIF(${CMAKE_C_COMPILER} MATCHES "gcc")



# These are all the places to search for header files which are
# to be part of the API.
SET(API_INCLUDE_DIRS) # start empty
FOREACH(subdir ${SIMBODY_SOURCE_SUBDIRS})
    # append
    SET(API_INCLUDE_DIRS ${API_INCLUDE_DIRS}
                         ${subdir}/include 
                         ${subdir}/include/simbody 
                         ${subdir}/include/simbody/internal)
ENDFOREACH(subdir)

# We'll need both *relative* path names, starting with their API_INCLUDE_DIRS,
# and absolute pathnames.
SET(API_REL_INCLUDE_FILES)   # start these out empty
SET(API_ABS_INCLUDE_FILES)

FOREACH(dir ${API_INCLUDE_DIRS})
    FILE(GLOB fullpaths ${dir}/*.h)	# returns full pathnames
    SET(API_ABS_INCLUDE_FILES ${API_ABS_INCLUDE_FILES} ${fullpaths})

    FOREACH(pathname ${fullpaths})
        GET_FILENAME_COMPONENT(filename ${pathname} NAME)
        SET(API_REL_INCLUDE_FILES ${API_REL_INCLUDE_FILES} ${dir}/${filename})
    ENDFOREACH(pathname)
ENDFOREACH(dir)

# collect up source files
SET(SOURCE_FILES) # empty
SET(SOURCE_INCLUDE_FILES)

FOREACH(subdir ${SIMBODY_SOURCE_SUBDIRS})
    FILE(GLOB src_files  ${subdir}/src/*.cpp ${subdir}/src/*/*.cpp)
    FILE(GLOB incl_files ${subdir}/src/*.h)
    SET(SOURCE_FILES         ${SOURCE_FILES}         ${src_files})   #append
    SET(SOURCE_INCLUDE_FILES ${SOURCE_INCLUDE_FILES} ${incl_files})

    ## Make sure we find these locally before looking in SimTK/include if
    ## Simbody was previously installed there.
    INCLUDE_DIRECTORIES(BEFORE ${PROJECT_SOURCE_DIR}/${subdir}/include)
ENDFOREACH(subdir)

INCLUDE_DIRECTORIES(BEFORE ${PROJECT_SOURCE_DIR}/src)

## Watch out -- this next line will cause CMake to find installed versions of
## simmatrix libraries before the local one. Be sure to delete installed ones first.
LINK_DIRECTORIES(${SimTK_SDK}/lib${LIB64})

IF (BUILD_TESTING)
    #
    # Allow automated build and dashboard.
    #
    INCLUDE (Dart)

    #IF (UNIX AND NOT CYGWIN AND NOT APPLE)
    #  IF (NOT CMAKE_BUILD_TYPE OR CMAKE_BUILD_TYPE MATCHES Debug)
    #    ADD_DEFINITIONS(-fprofile-arcs -ftest-coverage)
    #    LINK_LIBRARIES(gcov)
    #  ENDIF (NOT CMAKE_BUILD_TYPE OR CMAKE_BUILD_TYPE MATCHES Debug)
    #ENDIF (UNIX AND NOT CYGWIN AND NOT APPLE)

    #
    # Testing
    #
    ENABLE_TESTING()

ENDIF (BUILD_TESTING)

INCLUDE(ApiDoxygen.cmake)

#
# Installation
#
# CMake installs go into CMAKE_INSTALL_PREFIX, which is always
# set to something incorrect by CMake. We'll use SimTK_INSTALL_PREFIX
# instead which is passed in by the build system or set above.


SET(CMAKE_INSTALL_PREFIX ${SimTK_INSTALL_PREFIX} CACHE STRING "Install path prefix." FORCE)



# libraries and examples are installed from their subdirectories; headers here

# install headers
FILE(GLOB CORE_HEADERS     include/*.h                  */include/*.h)
FILE(GLOB TOP_HEADERS      include/simbody/*.h          */include/simbody/*.h)
FILE(GLOB INTERNAL_HEADERS include/simbody/internal/*.h */include/simbody/internal/*.h)
INSTALL_FILES(/include/                 FILES ${CORE_HEADERS})
INSTALL_FILES(/include/simbody/         FILES ${TOP_HEADERS})
INSTALL_FILES(/include/simbody/internal FILES ${INTERNAL_HEADERS})

# Notice that the (deprecated) INSTALL_FILES(dir ...) command works differently than the
# newer (recommended) INSTALL(dir FILES ...) command
# when the target directory starts with a slash
INSTALL( FILES ${PROJECT_SOURCE_DIR}/doc/SimbodyTheoryManual.pdf DESTINATION doc/simbody )

# These are at the end because we want them processed after
# all the various variables have been set above.

ADD_SUBDIRECTORY( staticTarget )
ADD_SUBDIRECTORY( sharedTarget )

IF (BUILD_VISUALIZER)
  SET(GUI_NAME "VisualizerGUI")
  ADD_SUBDIRECTORY(Visualizer/VisualizerGUI)
ENDIF (BUILD_VISUALIZER)

IF( BUILD_EXAMPLES )
  ADD_SUBDIRECTORY( examples )
ENDIF( BUILD_EXAMPLES )

IF( BUILD_TESTING )
  ADD_SUBDIRECTORY( tests )
ENDIF( BUILD_TESTING )
