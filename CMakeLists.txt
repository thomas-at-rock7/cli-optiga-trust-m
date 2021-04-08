cmake_minimum_required(VERSION "3.13.4")
project("cli-optiga-trust-m")

option(BUILD_FOR_RPI "BUILD_FOR_RPI" OFF) # Default
option(BUILD_FOR_ULTRA96 "BUILD_FOR_ULTRA96" OFF)
option(COLD_RESET "This is applicable if the host platform has GPIO option for RST and VDD." OFF) # Default
option(SOFT_RESET "This is applicable if the host platform doesn't have GPIO options for VDD and RST." OFF)
option(WARM_RESET "This is applicable if the host platform doesn't have GPIO option for VDD." OFF)


set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall")
add_definitions(-DENGINE_DYNAMIC_SUPPORT)

if (CMAKE_BUILD_TYPE)
    if (${CMAKE_BUILD_TYPE} STREQUAL "Release")
        message(STATUS "Build type Release")
    else()
        message(STATUS "Build type Debug")
        add_definitions(-DTRUSTM_ENGINE_DEBUG) # Enable TrustM debug messages
    endif()
else()
    message(STATUS "Build type not defined, defaulting to a Release build")
    set(CMAKE_BUILD_TYPE Release)
endif()

# Required libraries for linking
find_library(LIB_PTHREAD pthread)
find_library(LIB_RT rt)
find_library(LIB_SSL ssl)
find_library(LIB_CRYPTO crypto)

set (LIB_TRUSTM "trustm")
set (LIB_TRUSTM_ENGINE "trustm_engine")
set (TRUSTM "trustm_lib")
set (TRUSTM_HELPER "trustm_helper")
set (TRUSTM_UTIL "${TRUSTM}/optiga/util")
set (TRUSTM_CRYPT "${TRUSTM}/optiga/crypt")
set (TRUSTM_COMMS "${TRUSTM}/optiga/comms")
set (TRUSTM_COMMON "${TRUSTM}/optiga/common")
set (TRUSTM_CMD "${TRUSTM}/optiga/cmd")
set (PALDIR "${TRUSTM}/pal/linux")
set (ENGDIR "trustm_engine")
set (APPDIR "linux_example")
set (INCDIR $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/${TRUSTM}/optiga/include>
            $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/${TRUSTM}/optiga/include/optiga>
            $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/${TRUSTM}/optiga/include/optiga/ifx_i2c>
            $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/${TRUSTM}/optiga/include/optiga/comms>
            $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/${TRUSTM}/optiga/include/optiga/common>
            $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/${TRUSTM}/optiga/include/optiga/cmd>
            $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/${TRUSTM}/optiga/include/optiga/pal>
            $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/${TRUSTM}/pal/linux>
            $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/trustm_helper/include>
            $<INSTALL_INTERFACE:${TRUSTM}/optiga/include>
            $<INSTALL_INTERFACE:${TRUSTM}/optiga/include/optiga>
            $<INSTALL_INTERFACE:${TRUSTM}/optiga/include/optiga/ifx_i2c>
            $<INSTALL_INTERFACE:${TRUSTM}/optiga/include/optiga/comms>
            $<INSTALL_INTERFACE:${TRUSTM}/optiga/include/optiga/common>
            $<INSTALL_INTERFACE:${TRUSTM}/optiga/include/optiga/cmd>
            $<INSTALL_INTERFACE:${TRUSTM}/optiga/include/optiga/pal>
            $<INSTALL_INTERFACE:${TRUSTM}/pal/linux>
            $<INSTALL_INTERFACE:trustm_helper/include>
)
set (ENGINE_INC $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/${ENGDIR}>
            $<INSTALL_INTERFACE:${ENGDIR}>
)

# Set the reset type
if (COLD_RESET)
    message(STATUS "Cold reset defined")
    add_definitions(-DOPTIGA_COMMS_DEFAULT_RESET_TYPE=0)
elseif(SOFT_RESET)
    message(STATUS "Soft reset defined")
    add_definitions(-DOPTIGA_COMMS_DEFAULT_RESET_TYPE=1)
elseif(WARM_RESET)
    message(STATUS "Warm reset defined")
    add_definitions(-DOPTIGA_COMMS_DEFAULT_RESET_TYPE=2)
else()
    message(STATUS "Reset hasn't been defined a Cold reset will be used")
    add_definitions(-DOPTIGA_COMMS_DEFAULT_RESET_TYPE=0)
endif()

# Choose the appropiate i2c file depending on target, the default is rasberry pi
if (BUILD_FOR_RPI)
    message(STATUS "Using RPI i2c config")
    set (I2C_CONFIG "${PALDIR}/target/rpi3/pal_ifx_i2c_config.c")
elseif (BUILD_FOR_ULTRA96)
    message(STATUS "Using Ultra96 i2c config")
    set (I2C_CONFIG "${PALDIR}/target/ultra96/pal_ifx_i2c_config.c")
else()
    message(STATUS "Using Default I2C config which is RPI i2c config")
    set (I2C_CONFIG "${PALDIR}/target/rpi3/pal_ifx_i2c_config.c")
endif()

# Find all sources and build Trust M Library
file(GLOB_RECURSE HELPER_SOURCES LIST_DIRECTORIES true "${TRUSTM_HELPER}/*.c")
file(GLOB_RECURSE UTIL_SOURCES LIST_DIRECTORIES true "${TRUSTM_UTIL}/*.c")
file(GLOB_RECURSE CRYPT_SOURCES LIST_DIRECTORIES true "${TRUSTM_CRYPT}/*.c")
file(GLOB_RECURSE COMMS_SOURCES LIST_DIRECTORIES true "${TRUSTM_COMMS}/*.c")
file(GLOB_RECURSE COMMON_SOURCES LIST_DIRECTORIES true "${TRUSTM_COMMON}/*.c")
file(GLOB_RECURSE CMD_SOURCES LIST_DIRECTORIES true "${TRUSTM_CMD}/*.c")
add_library(${LIB_TRUSTM} SHARED
            ${HELPER_SOURCES}
            ${UTIL_SOURCES}
            ${CRYPT_SOURCES}
            ${COMMS_SOURCES}
            ${COMMON_SOURCES}
            ${CMD_SOURCES}
            ${PALDIR}/pal.c
            ${PALDIR}/pal_gpio.c
            ${PALDIR}/pal_i2c.c
            ${PALDIR}/pal_logger.c
            ${PALDIR}/pal_os_datastore.c
            ${PALDIR}/pal_os_event.c
            ${PALDIR}/pal_os_lock.c
            ${PALDIR}/pal_os_timer.c
            ${PALDIR}/pal_os_memory.c
            ${TRUSTM}/pal/pal_crypt_openssl.c
            ${I2C_CONFIG}
)
target_include_directories (${LIB_TRUSTM} PUBLIC "${INCDIR}")

# Find sources for Trust M library and build it
file(GLOB_RECURSE ENGINE_SOURCES LIST_DIRECTORIES true "${ENGDIR}/*.c")
add_library(${LIB_TRUSTM_ENGINE} SHARED ${ENGINE_SOURCES})
target_link_libraries(${LIB_TRUSTM_ENGINE} PUBLIC
                                ${LIB_TRUSTM}
                                ${LIB_PTHREAD}
                                ${LIB_RT}
                                ${LIB_SSL}
                                ${LIB_CRYPTO}
)
target_include_directories (${LIB_TRUSTM_ENGINE} PUBLIC "${ENGINE_INC}")

# Build all Trust M applications but don't install them
file(GLOB_RECURSE APP_SOURCES LIST_DIRECTORIES true "${APPDIR}/*.c")
foreach(appsrc IN ITEMS ${APP_SOURCES})
    get_filename_component(appbin ${appsrc} NAME_WE)
    add_executable(${appbin} ${appsrc})
    target_link_libraries(${appbin} PUBLIC
                            ${LIB_TRUSTM}
                            ${LIB_PTHREAD}
                            ${LIB_RT}
                            ${LIB_SSL}
                            ${LIB_CRYPTO}
    )
    target_include_directories (${appbin} PUBLIC "${INCDIR}")
endforeach()


# Installation, just install libraries
set(LIB_DEST lib)
set(ENGINE_LIB_DEST lib/engines-1.1)
install(TARGETS ${LIB_TRUSTM}
    EXPORT ${CMAKE_PROJECT_NAME}-targets
    ARCHIVE DESTINATION ${LIB_DEST}
    LIBRARY DESTINATION ${LIB_DEST}
)

install(TARGETS ${LIB_TRUSTM_ENGINE}
    EXPORT ${CMAKE_PROJECT_NAME}-targets
    ARCHIVE DESTINATION ${ENGINE_LIB_DEST}
    LIBRARY DESTINATION ${ENGINE_LIB_DEST}
)