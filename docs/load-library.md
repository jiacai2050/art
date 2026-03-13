# Android Runtime (ART) `nativeLoad` 实现分析

在 Android ART 中，`java.lang.Runtime.nativeLoad` 方法的 native 实现遵循以下调用链，涉及 OpenJDK 适配层、ART 核心运行时以及底层的 native 加载器。

## 调用链概览

1.  **Java 层**: `java.lang.Runtime.nativeLoad(String filename, ClassLoader loader, Class<?> caller)`
2.  **JNI 适配层 (OpenJDK)**: `JVM_NativeLoad` (位于 `art/openjdkjvm/OpenjdkJvm.cc`)
3.  **ART 核心运行时**: `art::JavaVMExt::LoadNativeLibrary` (位于 `art/runtime/jni/java_vm_ext.cc`)
4.  **底层加载器**: `android::OpenNativeLibrary` (位于 `art/libnativeloader/native_loader.cpp`)
5.  **系统调用**: `dlopen` (或通过 Native Bridge 加载)

---

## 详细实现

### 1. JNI 适配层：`JVM_NativeLoad`

由于 Android 采用了 OpenJDK 的核心库实现，`Runtime.java` 中的 `native` 方法通常映射到符合 JVM 规范的 `JVM_` 前缀函数。

**代码位置**: `art/openjdkjvm/OpenjdkJvm.cc`

```cpp
JNIEXPORT jstring JVM_NativeLoad(JNIEnv* env,
                                 jstring javaFilename,
                                 jobject javaLoader,
                                 jclass caller) {
  ScopedUtfChars filename(env, javaFilename);
  if (filename.c_str() == nullptr) {
    return nullptr;
  }

  std::string error_msg;
  {
    art::JavaVMExt* vm = art::Runtime::Current()->GetJavaVM();
    // 调用 ART 运行时的核心加载逻辑
    bool success = vm->LoadNativeLibrary(env,
                                         filename.c_str(),
                                         javaLoader,
                                         caller,
                                         &error_msg);
    if (success) {
      return nullptr;
    }
  }

  // 如果加载失败，清除可能存在的 JNI 异常并返回错误信息
  env->ExceptionClear();
  return env->NewStringUTF(error_msg.c_str());
}
```

### 2. ART 核心逻辑：`JavaVMExt::LoadNativeLibrary`

这是 ART 处理 native 库加载的核心函数。它负责检查 JNI 规范约束（如同一个库不能被多个 ClassLoader 加载）、管理已加载库的缓存以及触发 `JNI_OnLoad`。

**代码位置**: `art/runtime/jni/java_vm_ext.cc`

**核心步骤**:
1.  **检查缓存**: 在 `libraries_` 中查找是否已加载该路径的库。
2.  **验证 ClassLoader**: 如果已加载，检查其关联的 `ClassLoader` 是否与当前一致。
3.  **调用底层加载器**: 使用 `android::OpenNativeLibrary` 获取库句柄（handle）。
4.  **处理 Native Bridge**: 如果需要（例如在 ARM 设备上运行 x86 库），通过 Native Bridge 转换。
5.  **注册并回调**: 将库添加到已加载列表，并查找执行 `JNI_OnLoad`。

```cpp
bool JavaVMExt::LoadNativeLibrary(JNIEnv* env,
                                  const std::string& path,
                                  jobject class_loader,
                                  jclass caller_class,
                                  std::string* error_msg) {
  // ... (省略缓存检查逻辑) ...

  // 调用 libnativeloader 处理复杂的命名空间和 dlopen 逻辑
  void* handle = android::OpenNativeLibrary(
      env,
      runtime_->GetTargetSdkVersion(),
      path_str,
      class_loader,
      caller_location_str,
      library_path.get(),
      &needs_native_bridge,
      &nativeloader_error_msg);

  if (handle == nullptr) {
    // 处理错误...
    return false;
  }

  // ... (创建 SharedLibrary 对象并缓存) ...

  // 查找并调用 JNI_OnLoad
  void* sym = library->FindSymbol("JNI_OnLoad", nullptr, android::kJNICallTypeRegular);
  if (sym != nullptr) {
    using JNI_OnLoadFn = int(*)(JavaVM*, void*);
    JNI_OnLoadFn jni_on_load = reinterpret_cast<JNI_OnLoadFn>(sym);
    int version = (*jni_on_load)(this, nullptr);
    // ... (验证 JNI 版本) ...
  }

  return was_successful;
}
```

### 3. 底层加载：`libnativeloader`

`android::OpenNativeLibrary` 封装了 Android 复杂的链接器命名空间（Linker Namespaces）逻辑，确保应用只能访问其权限范围内的库。

**代码位置**: `art/libnativeloader/native_loader.cpp`

该层最终会根据当前环境决定调用：
-   标准的 `dlopen`
-   或者通过 `NativeBridge` 加载异构指令集的库。

## 关键点总结

-   **隔离性**: ART 严格遵循 JNI 规范，通过 `JavaVMExt` 确保 native 库与 `ClassLoader` 的绑定关系。
-   **兼容性**: 通过 `openjdkjvm` 层实现了与 OpenJDK 核心库的无缝对接。
-   **安全性**: 利用 `libnativeloader` 实现 Android 的加载器命名空间隔离，防止应用随意加载系统私有库。
