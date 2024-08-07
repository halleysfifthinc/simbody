#---------------------------------------------------
# Simmath
#
# Creates SimTK Core library, base name=SimTKmath.
# Default libraries are shared & optimized. Variants
# are created for static (_static) and debug (_d) and
# provision is made for an optional "namespace" (ns)
# and version number (vn).
#
# Windows:
#   [ns_]SimTKmath[_vn][_d].dll
#   [ns_]SimTKmath[_vn][_d].lib
#   [ns_]SimTKmath[_vn]_static[_d].lib
# Unix:
#   lib[ns_]SimTKmath[_vn][_d].so
#   lib[ns_]SimTKmath[_vn]_static[_d].a
#
# All libraries are installed in
#   %ProgramFiles%\SimTK\lib  (Windows)
#   /usr/local/SimTK/lib        (UNIX)
#
#----------------------------------------------------

project(SimTKmath VERSION ${Simbody_VERSION})

# SimTKmath depends on PlatformFiles and SimTKcommon only.
include_directories(${PLATFORM_INCLUDE_DIRECTORIES}
                ${SimTKCOMMON_INCLUDE_DIRECTORIES})

# code is in "./src" and in each of these subdirectories/src
set(SIMMATH_SOURCE_SUBDIRS LinearAlgebra Integrators Optimizers Geometry)

# Collect up information about the version of the SimTKmath library
# we're building and make it available to the code so it can be built
# into the binaries. This also determines the versioned library names
# in which case all dependencies must use the same version.

# This is the suffix if we're building and depending on versioned libraries.
set(VN "_${SimTKmath_VERSION}")

set(SIMMATH_COPYRIGHT_YEARS "2005-10")

# underbar separated list of dotted authors, no spaces or commas
set(SIMMATH_AUTHORS         "Jack.Middleton_Michael.Sherman")

add_compile_definitions(
    SimTK_SIMMATH_LIBRARY_NAME=${SimTKMATH_LIBRARY_NAME}
    SimTK_SIMMATH_MAJOR_VERSION=${SimTKmath_VERSION_MAJOR}
    SimTK_SIMMATH_MINOR_VERSION=${SimTKmath_VERSION_MINOR}
    SimTK_SIMMATH_PATCH_VERSION=${SimTKmath_VERSION_PATCH}
    SimTK_SIMMATH_COPYRIGHT_YEARS="${SIMMATH_COPYRIGHT_YEARS}"
    SimTK_SIMMATH_AUTHORS="${SIMMATH_AUTHORS}")

# -DSimTK_SIMMATH_TYPE has to be defined in the target subdirectories.
# -Dsimmath_EXPORTS defined automatically when Windows DLL build is being done.

set(SHARED_TARGET ${SimTKMATH_LIBRARY_NAME})
set(STATIC_TARGET ${SimTKMATH_LIBRARY_NAME}_static)
set(SHARED_TARGET_VN ${SimTKMATH_LIBRARY_NAME}${VN})
set(STATIC_TARGET_VN ${SimTKMATH_LIBRARY_NAME}${VN}_static)

## Test against the unversioned libraries if they are being built;
## otherwise against the versioned libraries.
if(BUILD_UNVERSIONED_LIBRARIES)
    set(TEST_SHARED_TARGET ${SHARED_TARGET})
    set(TEST_STATIC_TARGET ${STATIC_TARGET})
else()
    set(TEST_SHARED_TARGET ${SHARED_TARGET_VN})
    set(TEST_STATIC_TARGET ${STATIC_TARGET_VN})
endif()

# These are all the places to search for header files which are
# to be part of the API.
set(API_INCLUDE_DIRS) # start empty
set(SimTKMATH_INCLUDE_DIRS) # start empty
foreach(subdir . ${SIMMATH_SOURCE_SUBDIRS})
    list(APPEND API_INCLUDE_DIRS
     ${PROJECT_SOURCE_DIR}/${subdir}/include
     ${PROJECT_SOURCE_DIR}/${subdir}/include/simmath
     ${PROJECT_SOURCE_DIR}/${subdir}/include/simmath/internal)

    # Referencing headers must always be done relative to this level.
    list(APPEND SimTKMATH_INCLUDE_DIRS
     ${PROJECT_SOURCE_DIR}/${subdir}/include)
endforeach()

# Include the Simmath API include directories now so that Simmath code
# can use them.
include_directories(${SimTKMATH_INCLUDE_DIRS})

# And pass API include directories up to the parent so subsequent libraries
# can find the headers too.
set(SimTKMATH_INCLUDE_DIRECTORIES ${SimTKMATH_INCLUDE_DIRS}
    PARENT_SCOPE)


# We'll need both *relative* path names, starting with
# their API_INCLUDE_DIRS, and absolute pathnames.
set(API_REL_INCLUDE_FILES)   # start these out empty
set(API_ABS_INCLUDE_FILES)

foreach(dir ${API_INCLUDE_DIRS})
    file(GLOB fullpaths ${dir}/*.h)    # returns full pathnames
    list(APPEND API_ABS_INCLUDE_FILES ${fullpaths})

    foreach(pathname ${fullpaths})
        get_filename_component(filename ${pathname} NAME)
        list(APPEND API_REL_INCLUDE_FILES
        ${dir}/${filename})
    endforeach()
endforeach()

# collect up source files
set(SOURCE_FILES) # empty
set(SOURCE_INCLUDE_FILES) # for dependency tracking only, I believe (sherm)
set(SOURCE_INCLUDE_DIRS) # in case some low-level source needs a -I include dir
set(SOURCE_GROUPS)
set(SOURCE_GROUP_FILES)

# first process all the source subdirectories
foreach(subdir ${SIMMATH_SOURCE_SUBDIRS})
    add_subdirectory(${subdir})
endforeach()

if(SOURCE_GROUPS)
    list(LENGTH SOURCE_GROUPS NGROUPS)
    math(EXPR lastgrpnum ${NGROUPS}-1)
    foreach( grpnum RANGE 0 ${lastgrpnum} )
        list(GET SOURCE_GROUPS ${grpnum} grp)
        list(GET SOURCE_GROUP_FILES ${grpnum} grpfile)
        source_group("${grp}" FILES "${grpfile}")
    endforeach()
endif()

# then process ./src
file(GLOB src_files  ./src/*.cpp)
file(GLOB incl_files ./src/*.h)
list(APPEND SOURCE_FILES         ${src_files})
list(APPEND SOURCE_INCLUDE_FILES ${incl_files})

# Add low-level source include directories if any.
include_directories(${SOURCE_INCLUDE_DIRS})

#
# Installation
#

# libraries are installed from their subdirectories; headers here

# install headers
file(GLOB CORE_HEADERS     include/*.h                  */include/*.h)
file(GLOB TOP_HEADERS      include/simmath/*.h          */include/simmath/*.h)
file(GLOB INTERNAL_HEADERS include/simmath/internal/*.h */include/simmath/internal/*.h)

install(FILES ${CORE_HEADERS} DESTINATION ${SIMBODY_INCLUDE_INSTALL_DIR})
install(FILES ${TOP_HEADERS} DESTINATION ${SIMBODY_INCLUDE_INSTALL_DIR}/simmath)
install(FILES ${INTERNAL_HEADERS} DESTINATION ${SIMBODY_INCLUDE_INSTALL_DIR}/simmath/internal)

file(GLOB SIMMATH_DOCS doc/*.pdf doc/*.txt doc/*.md)
install(FILES ${SIMMATH_DOCS} DESTINATION ${CMAKE_INSTALL_DOCDIR})

# These are at the end because we want them processed after
# all the various variables have been set above.

if(BUILD_STATIC_LIBRARIES)
    add_subdirectory( staticTarget )
endif()
if(BUILD_DYNAMIC_LIBRARIES)
    add_subdirectory( sharedTarget )
endif()

if( BUILD_TESTING )
    add_subdirectory( tests )
endif()
