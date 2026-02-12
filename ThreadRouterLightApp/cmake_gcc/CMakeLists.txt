# Define minimal required version of CMake.
cmake_minimum_required(VERSION "3.25")

# Project definition
project(
	lighting-app
	VERSION 1.0
	LANGUAGES C CXX ASM
)

# Include the definition of the slc_lighting-app target,
# which contains the content of the SLC project
include(lighting-app.cmake)

add_executable(lighting-app
    # Add additional sources here
)

target_include_directories(lighting-app PUBLIC
    # Add additional include paths here
)

target_compile_definitions(lighting-app PUBLIC
    # Add additional macros here
)

target_compile_options(lighting-app PUBLIC
    # Set additional compiler flags here
)

target_link_options(lighting-app PUBLIC
    # Set additional linker flags here
)

# Link with the content defined in the SLC project
target_link_libraries(lighting-app PRIVATE
    slc
)

# Include managed project content if available
include(lighting-app_project.cmake OPTIONAL RESULT_VARIABLE managed_project)
if(managed_project)
    message(STATUS "Using managed project content from ${managed_project}")
endif()

# Force the gcc linker command
set_target_properties(lighting-app PROPERTIES LINKER_LANGUAGE C)

# Create .bin, .hex and .s37 artifacts after building the project
add_custom_command(TARGET lighting-app
    POST_BUILD
    COMMAND ${CMAKE_OBJCOPY} ${OBJCOPY_SREC_CMD} "$<TARGET_FILE:lighting-app>" "$<TARGET_FILE_DIR:lighting-app>/$<TARGET_FILE_BASE_NAME:lighting-app>.s37"
    COMMAND ${CMAKE_OBJCOPY} ${OBJCOPY_IHEX_CMD} "$<TARGET_FILE:lighting-app>" "$<TARGET_FILE_DIR:lighting-app>/$<TARGET_FILE_BASE_NAME:lighting-app>.hex"
    COMMAND ${CMAKE_OBJCOPY} ${OBJCOPY_BIN_CMD}  "$<TARGET_FILE:lighting-app>" "$<TARGET_FILE_DIR:lighting-app>/$<TARGET_FILE_BASE_NAME:lighting-app>.bin" 
)

# Run post-build pipeline to perform additional post-processing
if(post_build_command)
add_custom_command(TARGET lighting-app
    POST_BUILD
    WORKING_DIRECTORY ${CMAKE_CURRENT_LIST_DIR}/..
    COMMAND ${post_build_command}
)
endif()