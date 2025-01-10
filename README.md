# KRITIS³M WolfSSL Wrapper

This repository is a simple wrapper to ease compilation of [wolfssl](https://github.com/Laboratory-for-Safe-and-Secure-Systems/wolfssl) and the optional dependency [liboqs](https://github.com/open-quantum-safe/liboqs) for the KRITIS³M research project.

## Building

The project uses CMake to be built.

```bash
mkdir build && cd build
cmake [options] ..
make
sudo make install
```

You can also use Ninja as a build tool by specifying `-GNinja` within the CMake invocation.

The wrapper has the two libraries [wolfssl](https://github.com/Laboratory-for-Safe-and-Secure-Systems/wolfssl) and [liboqs](https://github.com/open-quantum-safe/liboqs) as dependencies. By default, those are cloned using the CMake FetchContent functionality. However, you can also specify their source directory via CMake variables (given below for each dependency) to prevent additional downloads.

* [wolfssl](https://github.com/Laboratory-for-Safe-and-Secure-Systems/wolfssl): KRITIS³M fork of WolfSSL with downstream changes (`-DFETCHCONTENT_SOURCE_DIR_WOLFSSL=/path/to/wolfssl`).
* [liboqs](https://github.com/open-quantum-safe/liboqs): Library for support of the PQC algorithm FALCON (`-DFETCHCONTENT_SOURCE_DIR_LIBOQS=/path/to/liboqs`).

The two compiled libraries are installed in the default CMake installation paths. Another install path may be specified via the default CMake `CMAKE_INSTALL_PREFIX` variable.

### CLI build options

The following additional CMake options are available to customize the compilation of the two libraries:
* `BUILD_SHARED_LIBS`: Select between shared libraries (.so/.dll) and static libraries (.a/.lib). Default: `ON`.
* `KRITIS3M_WOLFSSL_STANDALONE`: When this option is enabled, both WolfSSL and liboqs will be built as standalone librarys to be installed system-wide. When disabled, the libraries will be built as libraries only to be linked against a wrapping application. Default: `ON`.
* `KRITIS3M_WOLFSSL_ASM`: When this option is enabled, the WolfSSL library will be built with assembly optimizations enabled. This is only available for x86 on Linux and some ARM CPUs. Default: `ON`.
* `KRITIS3M_WOLFSSL_AARCH64_CRYPTO_EXT`: When this option is enabled, the WolfSSL library will be built using ARM cryptographic instructions. This only works if the given CPU supports these. Default: `ON`.
* `KRITIS3M_WOLFSSL_ENABLE_FALCON`: When this option is enabled, support for the PQC signature algorithm FALCON (FN-DSA) is enabled via the additional library liboqs. When disabled, the library will not be built. Default: `ON`.

## WolfSSL configuration

WolfSSL is configured using the `user_settings.h` header file (**not** via autoconf!). This header is generated during the CMake invocation depending on the CMake options from the provided `user_settings.h.in` file.

This header file is also installed system-wide during the install step to make sure consuming applications of WolfSSL find the configuration. You should be able to use WolfSSL in the same way as if it was compiled using autoconf.
