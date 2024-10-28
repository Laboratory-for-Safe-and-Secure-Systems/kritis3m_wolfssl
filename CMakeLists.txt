cmake_minimum_required(VERSION 3.22)

project(kritis3m_wolfssl)

include(GNUInstallDirs)
include(FetchContent)
set(FETCHCONTENT_QUIET FALSE)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# Necessary to allow FetchContent_Populate
if(POLICY CMP0169)
        cmake_policy(SET CMP0169 OLD)
endif()


# Options

# Select between shared or static libraries
option(BUILD_SHARED_LIBS "Build shared libraries (.so) instead of static ones (.a)" ON)

# When this option is enabled, both WolfSSL and liboqs will be built as standalone
# librarys to be installed system-wide. When disabled, the libraries will be
# built as libraries only to be linked against a wrapping application.
option(KRITIS3M_WOLFSSL_STANDALONE "Use standalone WolfSSL" ON)

# When this option is enabled, the WolfSSL library will be built with assembly optimizations
# enabled. This is only available for x86 on Linux and some ARM CPUs.
option(KRITIS3M_WOLFSSL_ASM "Enable assembly optimizations for WolfSSL" ON)


# External repositories we need
FetchContent_Declare(liboqs
        GIT_REPOSITORY          https://github.com/open-quantum-safe/liboqs.git
        GIT_TAG                 origin/main
        GIT_PROGRESS            TRUE
        USES_TERMINAL_DOWNLOAD  TRUE
)
FetchContent_Declare(wolfssl
        GIT_REPOSITORY          git@github.com:Laboratory-for-Safe-and-Secure-Systems/wolfssl.git
        GIT_TAG                 origin/development
        GIT_PROGRESS            TRUE
        USES_TERMINAL_DOWNLOAD  TRUE
)


if(MSVC)
        set(CMAKE_MSVC_RUNTIME_LIBRARY "")
        # if(BUILD_SHARED_LIBS)
        #         set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>DLL")
        # else()
        #         set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
        # endif()
        set(WARNING_C_FLAGS "")
endif()


# Liboqs configuration
set(OQS_DIST_BUILD  OFF)
set(OQS_BUILD_ONLY_LIB ON)
set(OQS_USE_OPENSSL OFF)
set(OQS_ENABLE_KEM_BIKE OFF)
set(OQS_ENABLE_KEM_FRODOKEM OFF)
set(OQS_ENABLE_KEM_NTRUPRIME OFF)
set(OQS_ENABLE_KEM_CLASSIC_MCELIECE OFF)
set(OQS_ENABLE_KEM_HQC OFF)
set(OQS_ENABLE_KEM_KYBER OFF)
set(OQS_ENABLE_KEM_ML_KEM OFF)
set(OQS_ENABLE_SIG_SPHINCS OFF)
set(OQS_ENABLE_SIG_DILITHIUM OFF)
set(OQS_ENABLE_SIG_MAYO OFF)
set(OQS_ENABLE_SIG_CROSS OFF)
set(OQS_ENABLE_SIG_ML_DSA OFF)
set(OQS_ENABLE_SIG_STFL_XMSS OFF)
set(OQS_ENABLE_SIG_STFL_LMS OFF)

# WolfSSL configuration
set(WOLFSSL_USER_SETTINGS ON)
set(WOLFSSL_EXAMPLES OFF)
set(WOLFSSL_CRYPT_TESTS OFF)
set(WOLFSSL_OQS OFF)
set(WOLFSSL_BUILD_OUT_OF_TREE ON)
set(WOLFSSL_X86_64_BUILD_ASM OFF)
if("${CMAKE_SYSTEM_PROCESSOR}" MATCHES "x86_64" AND KRITIS3M_WOLFSSL_ASM)
        message(STATUS "Enabling Intel ASM optimizations for WolfSSL")
        set(WOLFSSL_INTEL_ASM ON)
        set(WOLFSSL_X86_64_BUILD_ASM ON)
        set(WOLFSSL_AESNI ON)
elseif("${CMAKE_SYSTEM_PROCESSOR}" MATCHES "aarch64|arm64" AND KRITIS3M_WOLFSSL_ASM)
        message(STATUS "Enabling ARM ASM optimizations for WolfSSL")
        set(WOLFSSL_ARM_ASM ON)
endif()


# Actually add the external repositories to our project
FetchContent_GetProperties(liboqs)
if(NOT liboqs_POPULATED)
        FetchContent_Populate(liboqs)
        if(KRITIS3M_WOLFSSL_STANDALONE)
                add_subdirectory(${liboqs_SOURCE_DIR} ${liboqs_BINARY_DIR})
        else()
                add_subdirectory(${liboqs_SOURCE_DIR} ${liboqs_BINARY_DIR} EXCLUDE_FROM_ALL)
        endif()
endif()

FetchContent_GetProperties(wolfssl)
if(NOT wolfssl_POPULATED)
        FetchContent_Populate(wolfssl)
        if(KRITIS3M_WOLFSSL_STANDALONE)
                add_subdirectory(${wolfssl_SOURCE_DIR} ${wolfssl_BINARY_DIR})
        else()
                add_subdirectory(${wolfssl_SOURCE_DIR} ${wolfssl_BINARY_DIR} EXCLUDE_FROM_ALL)
        endif()
endif()

# Write the configuration into the user_settings.h file
configure_file(${CMAKE_CURRENT_SOURCE_DIR}/user_settings.h.in user_settings.h)

# In order for WolfSSL to find the 'user_settings.h' file, we have to
# provide the directory where it is located.
target_include_directories(wolfssl PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>)

# Provide the liboqs headers (generated during build) for WolfSSL
get_target_property(OQS_BINARY_DIR oqs BINARY_DIR)
target_include_directories(wolfssl PUBLIC $<BUILD_INTERFACE:${OQS_BINARY_DIR}/../include>)

# Link liboqs to WolfSSL
target_link_libraries(wolfssl PRIVATE oqs)

# Set additional compiler flags for WolfSSL
target_compile_options(wolfssl PRIVATE
        -march=native
        -mtune=native
)
if("${CMAKE_SYSTEM_PROCESSOR}" MATCHES "aarch64|arm64")
        target_compile_options(wolfssl PRIVATE -mstrict-align)
endif()


# Install our user_settings.h file
if (KRITIS3M_WOLFSSL_STANDALONE)
        install(FILES ${CMAKE_CURRENT_BINARY_DIR}/user_settings.h DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
endif()
