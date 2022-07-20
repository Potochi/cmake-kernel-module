cmake_minimum_required(VERSION 3.0 FATAL_ERROR)

project(<your-project-name-here> C ASM)

set(CMAKE_EXPORT_COMPILE_COMMANDS true)
set(kerneldir "" CACHE STRING "Path to the kernel build directory")

# Change this to your module name 
set(module_name "hello")

# Random definitions that I'm not sure are needed :^)
add_definitions(-D__KERNEL__ -DMODULE)

# Disallow in source builds
get_filename_component(srcdir "${CMAKE_SOURCE_DIR}" REALPATH)
get_filename_component(bindir "${CMAKE_BINARY_DIR}" REALPATH)

if(srcdir STREQUAL bindir)
  message(FATAL_ERROR "In source builds are not allowed. 
  Create a build directory and run cmake from there.")
endif()

if ("${kerneldir}" STREQUAL "")
  execute_process(COMMAND uname -r OUTPUT_VARIABLE uname_r OUTPUT_STRIP_TRAILING_WHITESPACE)
  set (kerneldir "/lib/modules/${uname_r}/build")
endif()

find_file(kernel_makefile NAMES Makefile PATHS ${kerneldir} NO_DEFAULT_PATH)
if (NOT kernel_makefile)
  message(FATAL_ERROR "Kernel makefile not found in ${kerneldir}")
endif()

# Find the kernel headers directory
# If you are using another kernel branch like 'linux-zen'
# you might need to change the directory in which the header
# file is searched
# Or you can set the kernel_headers_dir directly...
find_path(
        kernel_headers_dir
        include/linux/user.h
        PATHS /usr/src/linux-zen/
)
# ...like this
# set(kernel_headers_dir <path-to-kernel-headers>)
if (NOT kernel_headers_dir)
  message(FATAL_ERROR "Kernel headers not found. Modifications to CMakeLists.txt needed.")
endif()

# Add kernel include directories to compilation process
# This gives us juicy autocomplete
include_directories(${kernel_headers_dir}/include)

# For some reason on my machine building the kernel module does 
# not emit the 'compile_commands.json' file required for autocomplete
# This is why there is a dummy static library compiled.
# If you want you can use this to compile an actual static library
# that you can use for unit tests or whatever.
add_library(dummy-lib STATIC ".dummy.c")

# If you used the dummy library for something not 'dummy' you can
# uncomment the following line to add the library sources to the
# kernel module.
# get_target_property(module_sources dummy-lib SOURCES)
list(APPEND module_sources "${module_name}.c")

# Generate the list of source files that are going to be 
# converted to object files in the Kbuild file
string(REPLACE ";" " " module_sources_str "${module_sources}")
# Replace variables in Kbuild.in and output that to Kbuild
configure_file(Kbuild.in Kbuild @ONLY)

# Copy the source files to the directory cmake was configured in.
# This as to not complicate the makefile.
# TODO: Maybe find a better solution that would also allow for in-tree builds
foreach (src ${module_sources})
  configure_file(${src} ${src} COPYONLY)
endforeach()

# Add command to build the .ko file
set(module_cmd ${CMAKE_MAKE_PROGRAM} -C ${kerneldir} M=${CMAKE_CURRENT_BINARY_DIR})
add_custom_command(OUTPUT "${module_name}.ko"
  COMMAND ${module_cmd} modules
  WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
  DEPENDS ${module_sources} ${CMAKE_CURRENT_BINARY_DIR}/Kbuild
  VERBATIM)

# Add name agnostic build commands
add_custom_target(module DEPENDS "${module_name}.ko")
add_custom_target(module-clean COMMAND ${module_cmd} clean)
