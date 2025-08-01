cmake_minimum_required(VERSION 3.14)

# Get current date and time
string(TIMESTAMP BUILD_DATE "%Y-%m-%d")
string(TIMESTAMP BUILD_TIME "%H:%M:%S")
string(TIMESTAMP BUILD_TIMESTAMP "%Y-%m-%d %H:%M:%S")

# Configure build info header
configure_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/build_info.h.in
    ${CMAKE_CURRENT_BINARY_DIR}/build_info.h
    @ONLY
)

# Fetch nlohmann/json
include(FetchContent)
FetchContent_Declare(
    nlohmann_json
    GIT_REPOSITORY https://github.com/nlohmann/json.git
    GIT_TAG v3.11.3
)
FetchContent_MakeAvailable(nlohmann_json)

# Find Boost.ProgramOptions
find_package(Boost REQUIRED COMPONENTS program_options)
if(NOT Boost_FOUND)
    message(FATAL_ERROR "Boost.ProgramOptions is required. Please install:\n"
                        "  Ubuntu/Debian: sudo apt-get install libboost-program-options-dev\n"
                        "  macOS: brew install boost\n"
                        "  Windows: vcpkg install boost-program-options")
endif()

# V8 Console Application
add_executable(v8c 
    main.cpp
    DllLoader.cpp
    DllLoader.h
    V8Console.cpp
    V8Console.h
    V8ConsoleBuiltins.cpp
    V8ConsoleOutput.cpp
    V8ConsoleHelp.cpp
)

# Set properties
set_target_properties(v8c PROPERTIES
    CXX_STANDARD 20
    CXX_STANDARD_REQUIRED ON
    OUTPUT_NAME v8c
    RUNTIME_OUTPUT_DIRECTORY ${CMAKE_SOURCE_DIR}/Bin
)

# Link V8 and other dependencies
configure_v8_target(v8c)

# Link the console core library and nlohmann/json
target_link_libraries(v8c PRIVATE V8ConsoleCore nlohmann_json::nlohmann_json)

# Add compile definition based on USE_SYSTEM_V8
if(USE_SYSTEM_V8)
    target_compile_definitions(v8c PRIVATE USE_SYSTEM_V8)
endif()

# Option to disable readline (enabled by default)
option(USE_READLINE "Use GNU Readline for enhanced console features" ON)
option(AUTO_INSTALL_READLINE "Automatically install readline if missing (requires sudo)" OFF)

# Find readline if enabled
if(USE_READLINE)
    find_library(READLINE_LIB readline)
    find_path(READLINE_INCLUDE readline/readline.h)
    
    if(NOT READLINE_LIB OR NOT READLINE_INCLUDE)
        if(AUTO_INSTALL_READLINE)
            message(STATUS "GNU Readline not found. Attempting to install libreadline-dev...")
            
            # Check if we can use sudo
            execute_process(
                COMMAND sudo -n true
                RESULT_VARIABLE SUDO_RESULT
                OUTPUT_QUIET
                ERROR_QUIET
            )
            
            if(SUDO_RESULT EQUAL 0)
                # We have passwordless sudo
                message(STATUS "Installing libreadline-dev...")
                execute_process(
                    COMMAND sudo apt-get update -qq
                    RESULT_VARIABLE UPDATE_RESULT
                    OUTPUT_QUIET
                    ERROR_QUIET
                )
                execute_process(
                    COMMAND sudo apt-get install -y libreadline-dev
                    RESULT_VARIABLE INSTALL_RESULT
                    OUTPUT_VARIABLE INSTALL_OUTPUT
                    ERROR_VARIABLE INSTALL_ERROR
                )
                
                if(INSTALL_RESULT EQUAL 0)
                    message(STATUS "Successfully installed libreadline-dev")
                    # Clear the cache and try to find readline again
                    unset(READLINE_LIB CACHE)
                    unset(READLINE_INCLUDE CACHE)
                    find_library(READLINE_LIB readline)
                    find_path(READLINE_INCLUDE readline/readline.h)
                    
                    if(NOT READLINE_LIB OR NOT READLINE_INCLUDE)
                        message(FATAL_ERROR "Failed to find readline after installation. Please check your system.")
                    else()
                        message(STATUS "Found GNU Readline after installation: ${READLINE_LIB}")
                    endif()
                else()
                    message(WARNING "Failed to install libreadline-dev automatically: ${INSTALL_ERROR}")
                    message(FATAL_ERROR "Please install manually with: sudo apt-get install libreadline-dev\n"
                                        "Or disable readline support with -DUSE_READLINE=OFF")
                endif()
            else()
                # No passwordless sudo, prompt user
                message(STATUS "Cannot install libreadline-dev automatically (sudo requires password)")
                message(FATAL_ERROR "Please install libreadline-dev manually:\n"
                                    "  sudo apt-get install libreadline-dev\n"
                                    "Or disable readline support with -DUSE_READLINE=OFF")
            endif()
        else()
            # AUTO_INSTALL_READLINE is OFF
            message(FATAL_ERROR "GNU Readline not found. Please install libreadline-dev:\n"
                                "  sudo apt-get install libreadline-dev\n"
                                "Or disable readline support with -DUSE_READLINE=OFF")
        endif()
    else()
        message(STATUS "Found GNU Readline: ${READLINE_LIB}")
    endif()
else()
    message(STATUS "Building without GNU Readline support")
    target_compile_definitions(v8c PRIVATE NO_READLINE)
endif()

# Link Boost.ProgramOptions
if(Boost_FOUND)
    target_link_libraries(v8c PRIVATE Boost::program_options)
else()
    target_link_libraries(v8c PRIVATE Boost::program_options)
endif()

# Platform-specific linking
if(UNIX AND NOT APPLE)
    target_link_libraries(v8c PRIVATE dl)
    if(USE_READLINE AND READLINE_LIB)
        target_link_libraries(v8c PRIVATE ${READLINE_LIB})
    endif()
elseif(APPLE)
    target_link_libraries(v8c PRIVATE dl)
    if(USE_READLINE AND READLINE_LIB)
        target_link_libraries(v8c PRIVATE ${READLINE_LIB})
    endif()
elseif(WIN32)
    target_link_libraries(v8c PRIVATE kernel32)
endif()

# Add include directories
target_include_directories(v8c PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}
    ${CMAKE_CURRENT_BINARY_DIR}  # For generated build_info.h
    ${CMAKE_SOURCE_DIR}/Include
)

# Add third-party includes as SYSTEM to suppress warnings
target_include_directories(v8c SYSTEM PRIVATE
    ${CMAKE_SOURCE_DIR}/Include/third_party
    ${CMAKE_SOURCE_DIR}/External/rang/include
)

# Install target
install(TARGETS v8c
    RUNTIME DESTINATION bin
)