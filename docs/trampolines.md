# ART 中的 Trampolines (跳板/弹跳函数)

**Trampoline** 是计算机科学中的一个专业术语，指一段用于环境切换、地址重定向或调用约定转换的微型汇编指令片段。在 Android Runtime (ART) 中，它是衔接解释器、AOT 机器码、JIT 机器码和 Native (JNI) 函数的“胶水”。

## 1. 核心原理：为什么需要 Trampoline？

ART 是一个复杂的混合执行环境，一个方法在运行时可能处于多种状态。调用方（Caller）在跳转时往往不确定被调用方（Callee）的状态。

如果没有 Trampoline，每次函数调用都需要嵌入复杂的逻辑来判断：
*   目标方法是否已经加载？
*   目标方法是解释执行还是机器码执行？
*   目标方法是否是 JNI 函数？

Trampoline 的存在允许调用方直接跳转到一个预定义的“跳板”地址，由跳板在底层处理这些不确定性，从而保持生成代码的简洁与高效。

## 2. 三大核心场景深度解析

### 场景 A：Resolution Trampoline (解析跳板)
**解决问题：找不到地址（Routing）**

当 A 方法尝试调用 B 方法，但 B 尚未被类加载器解析（Link），或者 B 的内存地址尚未确定时使用。
*   **动作**: 
    1. 调用方跳转到 `art_quick_resolution_trampoline`。
    2. 跳板保存当前所有寄存器状态。
    3. 跳回 C++ 层的 `artQuickResolutionTrampoline` 函数。
    4. C++ 逻辑负责加载 B 类、解析方法并返回 B 的真正执行入口。
    5. 跳板恢复寄存器，并跳转到 B 的真正入口。

### 场景 B：Interpreter-to-Compiled Bridge (解释器到编译代码的桥梁)
**解决问题：数据格式不兼容（Data Conversion）**

解释器（Interpreter）使用 `Shadow Frame`（内存数组）存储变量，而 AOT/JIT 机器码使用 CPU 寄存器和硬件堆栈存储参数。
*   **动作**: 
    - 这是一个“搬运工”。它将 `Shadow Frame` 数组里的值按照目标平台的调用约定（如 ARM64 的 X0-X7 寄存器）一个个搬运到物理寄存器中。
    - 调整栈顶指针（SP），为机器码准备好物理堆栈环境。

### 场景 C：Generic JNI Trampoline (通用 JNI 跳板)
**解决问题：接口不适配（Language Interop / Marshaling）**

Java 的调用约定与 C/C++ 的标准调用约定（ABI）存在差异。为了避免为成千上万个 JNI 方法生成重复的适配代码，ART 提供了一个通用的汇编跳板。
*   **动作**: 
    - **参数映射**: 根据方法的签名（Shorty），动态地将 Java 参数搬运到 C 语言 ABI 要求的寄存器位置。
    - **状态切换**: 执行 `ThreadState` 切换。通知虚拟机该线程进入了 Native 模式，此时 GC 不会被该线程阻塞，但 GC 也不能移动该线程正在访问的 Java 对象。
    - **异常检查**: C 函数返回后，检查是否有悬挂的 Java 异常需要抛出。

## 3. 代码路径参考

*   **汇编实现 (Architecture-specific)**:
    - `runtime/arch/arm64/quick_entrypoints_arm64.S`
    - `runtime/arch/x86_64/quick_entrypoints_x86_64.S`
*   **C++ 逻辑入口**:
    - `runtime/entrypoints/quick/quick_trampoline_entrypoints.cc`
*   **编译器入口**:
    - `compiler/trampolines/trampoline_compiler.cc`

## 4. 总结对比

| 类型 | 术语 | 核心职责 | 隐喻 |
| :--- | :--- | :--- | :--- |
| **解析型** | `Resolution Trampoline` | 动态寻址与类加载 | **导航员**：负责带路 |
| **桥接型** | `Interpreter Bridge` | 影子栈帧与物理寄存器转换 | **翻译官**：转换数据格式 |
| **通用型** | `Generic JNI Trampoline` | Java 与 C 调用约定适配 | **适配器**：抹平语言接口差异 |
