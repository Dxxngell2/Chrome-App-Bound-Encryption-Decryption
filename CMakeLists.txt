cmake_minimum_required(VERSION 3.15)
project(ChromeABE LANGUAGES C CXX)

# Set output directories
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)

# Set C++ standard
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Options for build type
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release CACHE STRING "Build type" FORCE)
endif()

# Architecture selection (Win32/x64)
if(WIN32)
    if(NOT DEFINED CMAKE_GENERATOR_PLATFORM)
        set(CMAKE_GENERATOR_PLATFORM "x64" CACHE STRING "Platform (x64 or Win32)" FORCE)
    endif()
endif()

# Add sqlite3 static library
add_library(sqlite3 STATIC libs/sqlite/sqlite3.c)
target_include_directories(sqlite3 PUBLIC libs/sqlite)

# Add chacha include directory
include_directories(libs/chacha)

# Build encryptor EXE
add_executable(encryptor src/encryptor.cpp)
target_include_directories(encryptor PRIVATE libs/chacha)
target_link_libraries(encryptor PRIVATE sqlite3)
set_target_properties(encryptor PROPERTIES OUTPUT_NAME "encryptor")

# Build chrome_decrypt DLL
add_library(chrome_decrypt SHARED
    src/chrome_decrypt.cpp
    src/reflective_loader.c
)
target_include_directories(chrome_decrypt PRIVATE libs/sqlite)
target_link_libraries(chrome_decrypt PRIVATE sqlite3 bcrypt ole32 oleaut32 shell32 version comsuppw)
set_target_properties(chrome_decrypt PROPERTIES OUTPUT_NAME "chrome_decrypt" PREFIX "" SUFFIX ".dll")

# Custom command: run encryptor.exe to produce chrome_decrypt.enc
add_custom_command(
    OUTPUT ${CMAKE_BINARY_DIR}/chrome_decrypt.enc
    COMMAND encryptor $<TARGET_FILE:chrome_decrypt> ${CMAKE_BINARY_DIR}/chrome_decrypt.enc
    DEPENDS encryptor chrome_decrypt
    COMMENT "Encrypting chrome_decrypt.dll to chrome_decrypt.enc"
    VERBATIM
)

# Custom target for the encrypted payload
add_custom_target(encrypted_payload ALL
    DEPENDS ${CMAKE_BINARY_DIR}/chrome_decrypt.enc
)

# Add resource file to chrome_inject (Windows only)
if(WIN32)
    enable_language(RC)
    set(RESOURCE_FILE src/resource.rc)
    # Copy chrome_decrypt.enc to the source dir if needed for resource.rc
    add_custom_command(
        TARGET encrypted_payload POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
            ${CMAKE_BINARY_DIR}/chrome_decrypt.enc
            ${CMAKE_SOURCE_DIR}/chrome_decrypt.enc
        COMMENT "Copying chrome_decrypt.enc for resource embedding"
    )
    # chrome_inject depends on encrypted_payload (which ensures .enc is up to date)
    add_executable(chrome_inject
        src/chrome_inject.cpp
        src/syscalls.cpp
        ${RESOURCE_FILE}
    )
    add_dependencies(chrome_inject encrypted_payload)
    target_include_directories(chrome_inject PRIVATE libs/chacha)
    target_link_libraries(chrome_inject PRIVATE sqlite3 shell32 version)
    set_target_properties(chrome_inject PROPERTIES OUTPUT_NAME "chrome_inject")
else()
    # Non-Windows: build without resource
    add_executable(chrome_inject
        src/chrome_inject.cpp
        src/syscalls.cpp
    )
    add_dependencies(chrome_inject encrypted_payload)
    target_include_directories(chrome_inject PRIVATE libs/chacha)
    target_link_libraries(chrome_inject PRIVATE sqlite3 shell32 version)
    set_target_properties(chrome_inject PROPERTIES OUTPUT_NAME "chrome_inject")
endif()

# Post-build: copy DLL/EXE to root for compatibility with old script
add_custom_command(TARGET chrome_inject POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_if_different
        $<TARGET_FILE:chrome_inject>
        ${CMAKE_SOURCE_DIR}/chrome_inject.exe)
add_custom_command(TARGET chrome_decrypt POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_if_different
        $<TARGET_FILE:chrome_decrypt>
        ${CMAKE_SOURCE_DIR}/chrome_decrypt.dll)
add_custom_command(TARGET encryptor POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_if_different
        $<TARGET_FILE:encryptor>
        ${CMAKE_SOURCE_DIR}/encryptor.exe)

# Install targets (optional)
install(TARGETS chrome_inject chrome_decrypt encryptor
    RUNTIME DESTINATION bin
    LIBRARY DESTINATION bin
    ARCHIVE DESTINATION lib) 