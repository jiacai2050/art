# Android Runtime (ART)

## Project Overview

This is the source code for the Android Runtime (ART), the managed runtime used by applications and some system services on Android. ART replaces Dalvik as the runtime for Android 5.0 (Lollipop) and higher.

### Key Components

*   **Runtime (`runtime/`)**: The core runtime environment, including the garbage collector, thread management, and class linking.
*   **Compiler (`compiler/`)**: The ART compiler, responsible for Ahead-of-Time (AOT) and Just-in-Time (JIT) compilation of Dex bytecode.
*   **DalvikVM (`dalvikvm/`)**: The command-line utility to invoke the runtime.
*   **Dex2oat (`dex2oat/`)**: The AOT compiler driver.
*   **Tools (`tools/`)**: Various utilities like `dexdump`, `oatdump`, etc.
*   **Tests (`test/`)**: Comprehensive test suites (run-tests and gtests).

## Build System

ART uses the Android Build System (Soong), defined primarily by `Android.bp` files.

*   **Build Logic**: Located in `build/` and root `Android.bp`.
*   **Go Modules**: Build logic extensions are written in Go (e.g., `build/art.go`).

## Development & Testing

### Running Tests

The primary script for running tests is `art/test.py`.

#### Run-Tests (Data-driven Dex tests)
Located in `test/` (e.g., `test/001-HelloWorld`).

*   **Run all on host:**
    ```bash
    art/test.py --host -r
    ```
*   **Run specific test on host:**
    ```bash
    art/test.py --host -r -t 001-HelloWorld
    ```
*   **Run on target (device):**
    ```bash
    art/test.py --target -r
    ```

#### GTests (C++ Unit Tests)
Defined in `*_test.cc` files alongside source code.

*   **Run all on host:**
    ```bash
    art/test.py --host -g
    ```
*   **Run specific gtest module:**
    ```bash
    m test-art-host-gtest-art_runtime_tests
    ```

### Conventions

*   **Language**: Primarily C++17/C++20 (check specific flags in `build/`), Java, and Assembly.
*   **Code Style**: Follows AOSP C++ style guides.
*   **Directory Structure**: Source code is modularized (e.g., `runtime`, `compiler`, `libartbase`).
*   **License**: Apache 2.0 (mostly) and BSD.

## Useful Commands

*   **Help for test runner**:
    ```bash
    art/test.py -h
    ```
*   **List tools**:
    See `Android.bp` "art-tools" phony target for a list of available tools (e.g., `dexdump`, `hprof-conv`).
