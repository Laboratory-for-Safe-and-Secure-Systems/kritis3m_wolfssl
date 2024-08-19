cmake_minimum_required(VERSION 3.22)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

project(kritis3m_wolfssl)


# Options

# Select between shared or static libraries
option(BUILD_SHARED_LIBS "Build shared libraries (.so) instead of static ones (.a)" ON)

# When this option is enabled, both WolfSSL and liboqs will be built as standalone
# shared librarys to be installed system-wide. When disabled, the libraries will be
# built as static libraries to be linked against a wrapping application.
option(KRITIS3M_WOLFSSL_STANDALONE "Use standalone WolfSSL" ON)

# This option enables the necessary features for PKI support
option(KRITIS3M_WOLFSSL_PKI_SUPPORT "Enable PKI support in WolfSSL" OFF)


# External repositories we need
include(FetchContent)
Set(FETCHCONTENT_QUIET FALSE)
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
set(OQS_ENABLE_SIG_ML_DSA OFF)
set(OQS_ENABLE_SIG_STFL_XMSS OFF)
set(OQS_ENABLE_SIG_STFL_LMS OFF)

# WolfSSL configuration
set(WOLFSSL_USER_SETTINGS ON)
set(WOLFSSL_EXAMPLES OFF)
set(WOLFSSL_CRYPT_TESTS OFF)
set(WOLFSSL_OQS OFF)
set(WOLFSSL_BUILD_OUT_OF_TREE ON)
if("${CMAKE_SYSTEM_PROCESSOR}" MATCHES "x86_64")
        message(STATUS "Enabling Intel ASM optimizations for WolfSSL")
        set(WOLFSSL_INTEL_ASM ON)
# elseif("${CMAKE_SYSTEM_PROCESSOR}" MATCHES "aarch64|arm64")
#         message(STATUS "Enabling ARM ASM optimizations for WolfSSL")
#         set(WOLFSSL_ARM_ASM ON)
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

# In order for WolfSSL to find the 'user_settings.h' file, we have to
# provide the directory where it is located.
target_include_directories(wolfssl PUBLIC $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>)

# Provide the liboqs headers (generated during build) for WolfSSL
get_target_property(OQS_BINARY_DIR oqs BINARY_DIR)
target_include_directories(wolfssl PUBLIC $<BUILD_INTERFACE:${OQS_BINARY_DIR}/../include>)

# Link liboqs to WolfSSL
target_link_libraries(wolfssl PRIVATE oqs)


# Enable PKI support if requested
if(KRITIS3M_WOLFSSL_PKI_SUPPORT)
        target_compile_definitions(wolfssl PRIVATE KRITIS3M_WOLFSSL_PKI_SUPPORT)
endif()

