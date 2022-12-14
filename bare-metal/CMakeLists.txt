cmake_minimum_required(VERSION 3.16 FATAL_ERROR)
# this setting needs to be put forward to project() command
include(${CMAKE_SOURCE_DIR}/cmake/toolchain-Cortex-M7.cmake)

# set project property
project(oled LANGUAGES C ASM)
# set language standard and other property
set(CMAKE_C_STANDARD 11)
set(CMAKE_C_EXTENSIONS OFF)
set(CMAKE_C_STANDARD_REQUIRED ON)

# this setting can export compile_commands.json file for include path searching for VScode
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

message(STATUS "Using toolchain file: ${CMAKE_TOOLCHAIN_FILE}")

# Enable assembler files preprocessing
add_compile_options($<$<COMPILE_LANGUAGE:ASM>:-x$<SEMICOLON>assembler-with-cpp>)

add_compile_options(
    -Wall # print warning info
    -Wextra # print extra warning info
    -Wconversion # data conversion truncation warning
    -Wsign-conversion # sign <-> unsigned conversion warning
)

if ("${CMAKE_BUILD_TYPE}" STREQUAL "Release")
    message(STATUS "Maximum optimization for speed")
    add_compile_options(-Ofast)
elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "RelWithDebInfo")
    message(STATUS "Maximum optimization for speed, debug info included")
    add_compile_options(-Ofast -g)
elseif ("${CMAKE_BUILD_TYPE}" STREQUAL "MinSizeRel")
    message(STATUS "Maximum optimization for size")
    add_compile_options(-Os)
else ()
    message(STATUS "Minimal optimization, debug info included")
    add_compile_options(-Og -g)
endif ()

include_directories(
    ./
    Core/Inc
    Components/Inc
    Hardware/Inc
    Drivers/STM32F7xx_HAL_Driver/Inc
    Drivers/STM32F7xx_HAL_Driver/Inc/Legacy
    Drivers/CMSIS/Device/ST/STM32F7xx/Include
    Drivers/CMSIS/Include
)

file(GLOB_RECURSE SOURCES
    "startup/ARM/*.*"
    "Drivers/*.*"
    "Core/*.*"
    "Components/*.*"
    "Hardware/*.*"
)

set(LINKER_SCRIPT ${CMAKE_SOURCE_DIR}/STM32F767ZITx_FLASH.ld)

add_link_options(
    -T ${LINKER_SCRIPT}
    LINKER:--print-memory-usage
    LINKER:-Map=${PROJECT_BINARY_DIR}/${PROJECT_NAME}.map
)

add_executable(${PROJECT_NAME} ${SOURCES} ${LINKER_SCRIPT})

set_target_properties(${PROJECT_NAME} PROPERTIES
   SUFFIX .elf
   LINK_DEPENDS ${LINKER_SCRIPT}
)


if (EXISTS ${CMAKE_OBJCOPY})
    add_custom_command(
        TARGET ${PROJECT_NAME}
        POST_BUILD
        COMMAND ${CMAKE_OBJCOPY} -O ihex $<TARGET_FILE:${PROJECT_NAME}>
                ${CMAKE_BINARY_DIR}/$<TARGET_NAME:${PROJECT_NAME}>.hex
        COMMENT "Building $<TARGET_NAME:${PROJECT_NAME}>.hex"
    )

    add_custom_command(
        TARGET ${PROJECT_NAME}
        POST_BUILD
        COMMAND ${CMAKE_OBJCOPY} -O binary $<TARGET_FILE:${PROJECT_NAME}>
                ${CMAKE_BINARY_DIR}/$<TARGET_NAME:${PROJECT_NAME}>.bin
        COMMENT "Building $<TARGET_NAME:${PROJECT_NAME}>.bin"
    )

else()
    message(STATUS "'objcopy' not found: cannot generate .hex file")
endif()

if (EXISTS "${TOOLCHAIN_SIZE}")
    add_custom_command(
        TARGET ${PROJECT_NAME}
        POST_BUILD
        COMMAND ${TOOLCHAIN_SIZE} --format=berkeley $<TARGET_FILE:${PROJECT_NAME}>
                >${CMAKE_BINARY_DIR}/$<TARGET_NAME:${PROJECT_NAME}>.bsz
    )

    add_custom_command(
        TARGET ${PROJECT_NAME}
        POST_BUILD
        COMMAND ${TOOLCHAIN_SIZE} --format=sysv -x $<TARGET_FILE:${PROJECT_NAME}>
                >${CMAKE_BINARY_DIR}/$<TARGET_NAME:${PROJECT_NAME}>.ssz
    )
else()
    message(STATUS "'size' not found: cannot generate .[bs]sz files")
endif()