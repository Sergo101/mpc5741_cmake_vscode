cmake_minimum_required(VERSION 3.22)

#
# User is free to modify the file as much as necessary
#

list(APPEND CMAKE_MODULE_PATH "{{sr:cmake_path}}")
message("Build CMAKE_MODULE_PATH: " ${CMAKE_MODULE_PATH})
include("cmake/powerpc-eabivle.cmake")

find_program(PPC_GCC powerpc-eabivle-gcc)
message(${PPC_GCC})
if(NOT PPC_GCC)
    message(FATAL_ERROR "Cannot find powerpc-eabivle-gcc compiler. Specify path to powerpc-eabivle-gcc.exe in PATH system variable.")
endif()

# Get the folder for the C compiler
get_filename_component(COMPILER_DIR "${PPC_GCC}" DIRECTORY)

set(E200_root ${COMPILER_DIR}/../../e200_ewl2)

# Core project settings
project(cmake_mpc57_test)
enable_language(C CXX ASM)
message("Build type: " ${CMAKE_BUILD_TYPE})

# Setup compiler settings
set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS ON)
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS ON)

# Core MCU flags, CPU type, instruction set and FPU setup
set(cpu_PARAMS 
    -c
    -fmessage-length=0
    -fstrict-volatile-bitfields
    -ffunction-sections
    -fdata-sections
    -mcpu=e200z4
    -mbig
    -mvle
    -mregnames
    -mhard-float
)

# Sources
file(GLOB sources_SRCS
  ${CMAKE_CURRENT_LIST_DIR}/src/*.c
  ${CMAKE_CURRENT_LIST_DIR}/src/*.s
  ${CMAKE_CURRENT_LIST_DIR}/Project_Settings/Startup_Code/startup.S
)

# Include directories for all compilers
set(include_DIRS
    ${CMAKE_CURRENT_LIST_DIR}/include/
)

# Include directories for each compiler
set(include_c_DIRS)
set(include_cxx_DIRS)
set(include_asm_DIRS)

# Symbols definition for all compilers
set(symbols_SYMB
MPC574xP
)

# Symbols definition for each compiler
set(symbols_c_SYMB)
set(symbols_cxx_SYMB)
set(symbols_asm_SYMB)

# Link directories and names of libraries
set(link_DIRS   
)

set(link_LIBS
)

# Linker script
set(linker_script_SRC
-T${CMAKE_CURRENT_LIST_DIR}/Project_Settings/Linker_Files/57xx_flash.ld
-T${CMAKE_CURRENT_LIST_DIR}/Project_Settings/Linker_Files/57xx_ram.ld
)

# Compiler options
set(compiler_OPTS)

# Linker options
set(linker_OPTS)

# Now call generated cmake
# This will add script generated
# information to the project


# Link directories setup
# Must be before executable is added
link_directories(${CMAKE_PROJECT_NAME} ${link_DIRS})

# Create an executable object type
add_executable(${CMAKE_PROJECT_NAME})

# Add sources to executable
target_sources(${CMAKE_PROJECT_NAME} PUBLIC ${sources_SRCS})

# Add include paths
target_include_directories(${CMAKE_PROJECT_NAME} PRIVATE
    ${include_DIRS}
    $<$<COMPILE_LANGUAGE:C>: ${include_c_DIRS}>
    $<$<COMPILE_LANGUAGE:CXX>: ${include_cxx_DIRS}>
    $<$<COMPILE_LANGUAGE:ASM>: ${include_asm_DIRS}>
)

# Add project symbols (macros)
target_compile_definitions(${CMAKE_PROJECT_NAME} PRIVATE
    ${symbols_SYMB}
    $<$<COMPILE_LANGUAGE:C>: ${symbols_c_SYMB}>
    $<$<COMPILE_LANGUAGE:CXX>: ${symbols_cxx_SYMB}>
    $<$<COMPILE_LANGUAGE:ASM>: ${symbols_asm_SYMB}>

    # Configuration specific
    $<$<CONFIG:Debug>: >
    $<$<CONFIG:Release>: >
)

# Add linked libraries
target_link_libraries(${CMAKE_PROJECT_NAME} ${link_LIBS})

# Compiler options
target_compile_options(${CMAKE_PROJECT_NAME} PRIVATE
    ${cpu_PARAMS}
    ${compiler_OPTS}

    $<$<COMPILE_LANGUAGE:C>: >
    $<$<COMPILE_LANGUAGE:CXX>:

    # -Wno-volatile
    # -Wold-style-cast
    # -Wuseless-cast
    # -Wsuggest-override
    >
    $<$<COMPILE_LANGUAGE:ASM>:-x assembler-with-cpp -MMD -MP>
    $<$<CONFIG:Debug>:-Og -g3 -ggdb>
    $<$<CONFIG:Release>:-Og -g0>
)

# Linker options
target_link_options(${CMAKE_PROJECT_NAME} PRIVATE
    # ${linker_script_SRC}
    -T${CMAKE_CURRENT_LIST_DIR}/Project_Settings/Linker_Files/57xx_flash.ld
    # -T${CMAKE_CURRENT_LIST_DIR}/Project_Settings/Linker_Files/57xx_ram.ld
    -Wl,-Map=${CMAKE_PROJECT_NAME}.map
    -Xlinker
    --gc-sections
    -mcpu=e200z4
    -mhard-float
    --sysroot=${E200_root}
)

# Execute post-build to print size, generate hex and bin
add_custom_command(TARGET ${CMAKE_PROJECT_NAME} POST_BUILD
    COMMAND ${CMAKE_SIZE} $<TARGET_FILE:${CMAKE_PROJECT_NAME}>
    COMMAND ${CMAKE_OBJCOPY} -O ihex $<TARGET_FILE:${CMAKE_PROJECT_NAME}> ${CMAKE_PROJECT_NAME}.hex
    COMMAND ${CMAKE_OBJCOPY} -O binary $<TARGET_FILE:${CMAKE_PROJECT_NAME}> ${CMAKE_PROJECT_NAME}.bin
)
