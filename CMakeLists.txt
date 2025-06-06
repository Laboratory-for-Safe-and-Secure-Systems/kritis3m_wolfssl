cmake_minimum_required(VERSION 3.22)

project(kritis3m_wolfssl)

include(GNUInstallDirs)
include(FetchContent)
set(FETCHCONTENT_QUIET FALSE)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

enable_language(C ASM)

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

# When this option is enabled, the WolfSSL library will be built using ARM cryptographic
# instructions. This only works if the given CPU supports these.
option(KRITIS3M_WOLFSSL_AARCH64_CRYPTO_EXT "Use ARM Aarch64 cryptographic extensions" ON)

# When this option is enabled, support for the PQC signature algorithm FALCON (FN-DSA) is
# enabled via the additional library liboqs. When disabled, the library will not be built.
option(KRITIS3M_WOLFSSL_ENABLE_FALCON "Enable FALCON (FN-DSA) signature algorithm" ON)

# When enabled, heap memory tracking is enabled in WolfSSL.
option(KRITIS3M_WOLFSSL_HEAP_TRACKING "Enable heap memory tracking" OFF)


# Make sure all libraries are installed in the lib directory (not lib64)
set(CMAKE_INSTALL_LIBDIR "lib")


# External repositories we need
FetchContent_Declare(wolfssl
        GIT_REPOSITORY          git@github.com:Laboratory-for-Safe-and-Secure-Systems/wolfssl.git
        GIT_TAG                 origin/development
        GIT_PROGRESS            TRUE
        USES_TERMINAL_DOWNLOAD  TRUE
)
if(KRITIS3M_WOLFSSL_ENABLE_FALCON)
        FetchContent_Declare(liboqs
                GIT_REPOSITORY          https://github.com/open-quantum-safe/liboqs.git
                GIT_TAG                 origin/main
                GIT_PROGRESS            TRUE
                USES_TERMINAL_DOWNLOAD  TRUE
        )
endif()

if(MSVC)
        set(CMAKE_MSVC_RUNTIME_LIBRARY "")
        # if(BUILD_SHARED_LIBS)
        #         set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>DLL")
        # else()
        #         set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
        # endif()
        set(WARNING_C_FLAGS "")
endif()


if(KRITIS3M_WOLFSSL_ENABLE_FALCON)
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
        set(OQS_ENABLE_SIG_UOV OFF)
        set(OQS_ENABLE_SIG_SNOVA OFF)
        set(OQS_ENABLE_SIG_STFL_XMSS OFF)
        set(OQS_ENABLE_SIG_STFL_LMS OFF)
        set(OQS_ENABLE_SIG_ML_DSA OFF)
        set(OQS_ENABLE_SIG_STFL_XMSS OFF)
        set(OQS_ENABLE_SIG_STFL_LMS OFF)

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
endif()

# WolfSSL configuration
set(WOLFSSL_USER_SETTINGS ON)
set(WOLFSSL_EXAMPLES OFF)
set(WOLFSSL_CRYPT_TESTS OFF)
set(WOLFSSL_OQS OFF)
set(WOLFSSL_BUILD_OUT_OF_TREE ON)
set(WOLFSSL_X86_64_BUILD_ASM OFF)
set(WOLFSSL_PKCS11 ON)
set(WOLFSSL_CONFIG_H OFF)
if("${CMAKE_SYSTEM_PROCESSOR}" MATCHES "x86_64|AMD64" AND KRITIS3M_WOLFSSL_ASM)
        message(STATUS "Enabling Intel ASM optimizations for WolfSSL")
        set(WOLFSSL_INTEL_ASM ON)
        set(WOLFSSL_X86_64_BUILD_ASM ON)
        set(WOLFSSL_AESNI ON)
elseif("${CMAKE_SYSTEM_PROCESSOR}" MATCHES "aarch64|arm64" AND KRITIS3M_WOLFSSL_ASM)
        message(STATUS "Enabling ARM ASM optimizations for WolfSSL")
        set(WOLFSSL_ARM_ASM ON)
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


if(KRITIS3M_WOLFSSL_ENABLE_FALCON)
        # Provide the liboqs headers (generated during build) for WolfSSL
        get_target_property(OQS_BINARY_DIR oqs BINARY_DIR)
        target_include_directories(wolfssl PUBLIC $<BUILD_INTERFACE:${OQS_BINARY_DIR}/../include>)

        # Link liboqs to WolfSSL
        target_link_libraries(wolfssl PRIVATE oqs)
endif()

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
