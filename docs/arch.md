# Android Runtime (ART) 架构概览

Android Runtime (ART) 是 Android 操作系统的核心托管运行时，负责执行应用程序和系统服务的 Dex 字节码。

## 1. 整体架构 (Overall Architecture)

ART 的架构采用模块化设计，主要分为执行层、编译层和基础设施层：

*   **执行层 (Execution Layer)**：
    *   **解释器 (Interpreter)**：负责直接执行 Dex 字节码。现代 ART 包含 `nterp`（高性能汇编解释器），用于提升未编译代码的执行速度。
    *   **JIT (Just-In-Time) 编译器**：在运行时识别“热点方法”并将其动态编译为机器码。
*   **编译层 (Compilation Layer)**：
    *   **Optimizing Compiler**：核心编译器后端，支持多种优化（如内联、GVN、寄存器分配），并针对不同架构（ARM, X86, RISC-V）生成优化后的代码。
    *   **dex2oat**：AOT (Ahead-of-Time) 编译驱动程序，将 Dex 文件转换为包含机器码的 OAT 文件（本质上是 ELF 格式）。
*   **基础设施层 (Infrastructure Layer)**：
    *   **Runtime (`runtime/`)**：管理 VM 状态、内存分配、GC、线程同步和类链接。
    *   **libdexfile**：提供对 DEX 和 CompactDEX 格式的解析和访问接口。
    *   **libartbase**：底层基础库，包含容器、日志、内存映射等通用工具。

## 2. 项目工作流程 (Project Workflow)

ART 的执行逻辑遵循**混合执行模式**，旨在平衡安装速度、内存占用和运行性能：

1.  **编译期 (AOT/dex2oat)**：
    *   在应用安装或系统空闲时，`dex2oat` 工具运行。
    *   它读取 `.dex` 文件，根据**配置文件 (Profiles)** 仅编译常用的方法，生成 `.oat` 和 `.vdex` 文件。
2.  **启动与加载**：
    *   **Zygote**：系统启动时预加载 ART 运行时，通过 `fork()` 快速创建应用进程。
    *   **Class Linker**：负责寻找并加载类，验证字节码，并链接方法入口点。
3.  **运行期 (Hybrid Execution)**：
    *   **解释执行**：应用启动初期，不常用的代码通过解释器执行。
    *   **JIT 编译**：随着运行，JIT 记录热点方法，在后台将其编译为机器码并替换入口点。
    *   **垃圾回收 (GC)**：ART 的 GC（如 Concurrent Copying）在后台并发运行，最小化应用停顿 (Stop-the-world)。

## 3. 核心组件 (Core Components)

*   **Runtime (`runtime/`)**：
    *   `gc/`：实现了多种垃圾回收算法，目前主流是 **Concurrent Copying (CC)**。
    *   `class_linker.cc`：核心逻辑，负责类的解析、链接和方法查找。
    *   `thread.cc`：管理托管线程与本地线程的映射及同步。
*   **Compiler (`compiler/`)**：
    *   包含中间表示 (IR) 构建、优化 Pass 和各平台的后端代码生成器。
*   **dex2oat (`dex2oat/`)**：
    *   作为 AOT 编译的入口，它协调读取 Dex、调用编译器、并将结果打包成 OAT/ELF 格式。
*   **odrefresh (`odrefresh/`)**：
    *   现代 ART 引入的工具，用于在系统更新或环境变化时，确保 `/data` 分区中的核心类库（如 Boot Classpath）是最新的编译版本。
*   **artd (`artd/`)**：
    *   ART 守护进程，负责处理特权级的 DEX 优化任务和文件管理。
*   **sigchainlib (`sigchainlib/`)**：
    *   信号链库。ART 使用硬件信号（如 `SIGSEGV`）来优化 NullCheck 和隐式栈检查，该组件确保 ART 的信号处理不与应用中的 native 代码冲突。

## 4. 目录结构角色

*   `dalvikvm/`：命令行工具，用于在终端直接启动一个 ART 实例。
*   `libnativebridge/` / `libnativeloader/`：处理 Native 库的加载，支持跨架构的代码执行（如在 ARM 设备上运行 X86 代码）。
*   `openjdkjvmti/`：实现 Java 虚拟机工具接口，支持 Android Studio 的 Profiler 和 Debugger。
*   `test/`：包含大量的 `run-test`（功能测试）和 `gtest`（C++ 单元测试）。
