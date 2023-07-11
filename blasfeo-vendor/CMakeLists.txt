cmake_minimum_required(VERSION 3.1)

project(blasfeo)

if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release)
endif()

##############################
# MULTI-TARGET BUILD OPTIONS #
##############################

option(MULTI_X64_BUILD "Build BLASFEO for multiple x64 targets." OFF)

# Specify paths where to install the different target builds
set(BLASFEO_INSTALL_PREFIX "/opt/blasfeo" CACHE STRING "Install prefixes: ${BLASFEO_INSTALL_PREFIX}/${TARGET}.")

# Define BLASFEO targets to build
list(APPEND MULTI_ARCH_X64_TARGETS
    X64_INTEL_HASWELL
    X64_INTEL_CORE
    X64_INTEL_SANDY_BRIDGE
    X64_AMD_BULLDOZER
    GENERIC
)

#########################
# DEFAULT CONFIGURATION #
#########################

if(CMAKE_SYSTEM_PROCESSOR STREQUAL "x86_64")
    set(TARGET X64_AUTOMATIC CACHE STRING  "Default options for blasfeo.")
else()
    set(TARGET ARMV8A_ARM_CORTEX_A53 CACHE STRING  "Default options for blasfeo.")
endif()

set(BLASFEO_HEADERS_INSTALLATION_DIRECTORY  include/blasfeo CACHE STRING  "Default options for blasfeo.")
set(BLASFEO_EXAMPLES                        OFF             CACHE BOOL    "Default options for blasfeo.")
set(BUILD_SHARED_LIBS                       ON              CACHE BOOL    "Default options for blasfeo.")

###############
# BUILD LOGIC #
###############

if(NOT MULTI_X64_BUILD)
    add_subdirectory(external/blasfeo)
else()
    include(ExternalProject)

    # Obtain path of CMAKE modules
    get_filename_component(MULTI_TARGET_BUILD_MODULES "${CMAKE_CURRENT_SOURCE_DIR}/cmake/modules" ABSOLUTE)

    # Make paths absolute
    get_filename_component(BLASFEO_INSTALL_PREFIX_ABSOLUTE ${BLASFEO_INSTALL_PREFIX} ABSOLUTE)

    #############################
    # BUILD AND INSTALL BLASFEO #
    #############################

    # Dummy target for bootstrapping dependencies, necessary for parallel build
    add_custom_target(dummy_target)
    list(APPEND BUILD_ORDER dummy_target)

    message(STATUS "############## blasfeo targets ##############")
    foreach(BLASFEO_TARGET ${MULTI_ARCH_X64_TARGETS})
        message(STATUS "# ${BLASFEO_TARGET}")
        # Build
        ExternalProject_Add(blasfeo_${BLASFEO_TARGET}
            DEPENDS ${BUILD_ORDER}
            SOURCE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/external/blasfeo
            CMAKE_ARGS -DCMAKE_INSTALL_PREFIX=${CMAKE_CURRENT_BINARY_DIR}/blasfeo_install/${BLASFEO_TARGET}
                       -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
                       -DTARGET=${BLASFEO_TARGET}
                       -DBUILD_SHARED_LIBS=${BUILD_SHARED_LIBS}
                       -DBLASFEO_HEADERS_INSTALLATION_DIRECTORY=${BLASFEO_HEADERS_INSTALLATION_DIRECTORY}
                       -DBLASFEO_EXAMPLES=${BLASFEO_EXAMPLES}
        )
        list(APPEND BUILD_ORDER blasfeo_${BLASFEO_TARGET})
        list(REMOVE_AT BUILD_ORDER 0)

        # Install
        install(
            DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/blasfeo_install/${BLASFEO_TARGET}/lib/
            DESTINATION ${BLASFEO_INSTALL_PREFIX_ABSOLUTE}/${BLASFEO_TARGET}/lib
        )
        install(
            DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/blasfeo_install/${BLASFEO_TARGET}/include/
            DESTINATION ${BLASFEO_INSTALL_PREFIX_ABSOLUTE}/${BLASFEO_TARGET}/include
        )
        install(
            DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}/blasfeo_install/${BLASFEO_TARGET}/cmake/
            DESTINATION ${BLASFEO_INSTALL_PREFIX_ABSOLUTE}/${BLASFEO_TARGET}/cmake
        )
    endforeach()
    message(STATUS "#############################################")

    ############################
    # CONFIGURE DEFAULT TARGET #
    ############################

    if (BUILD_SHARED_LIBS)
        set(LIBFILETYPE "so")
    else ()
        set(LIBFILETYPE "a")
    endif() 
    configure_file(cmake/modules/configure_default_target.cmake.in 
        ${CMAKE_CURRENT_BINARY_DIR}/configure_default_target.cmake
        @ONLY
    )
    install(DIRECTORY DESTINATION ${CMAKE_INSTALL_PREFIX}/lib)
    install(DIRECTORY DESTINATION ${CMAKE_INSTALL_PREFIX}/include)
    install(DIRECTORY DESTINATION ${CMAKE_INSTALL_PREFIX}/cmake)
    install(SCRIPT ${CMAKE_CURRENT_BINARY_DIR}/configure_default_target.cmake)
endif()
