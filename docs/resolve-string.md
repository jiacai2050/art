# ART String Resolution (ResolveString)

## 入口点

1. **解释器** - `const-string` 指令
   - `runtime/interpreter/interpreter_switch_impl-inl.h::HandleConstString()`
2. **编译代码** - 运行时入口点 `pResolveString`
   - `runtime/entrypoints/quick/quick_dexcache_entrypoints.cc::artResolveStringFromCode()`
3. **nterp 解释器**
   - `runtime/interpreter/mterp/nterp.cc`
4. **反射/元数据** - 获取方法名、字段名
   - `runtime/art_method-inl.h`、`runtime/art_field-inl.h`

## 核心调用链

```
ClassLinker::ResolveString(string_idx, referrer)       // runtime/class_linker-inl.h
  │
  ├─ DexCache::GetResolvedString(string_idx)           // runtime/mirror/dex_cache-inl.h
  │    └─ 命中缓存 → 直接返回
  │
  └─ (未命中) ClassLinker::DoResolveString()           // runtime/class_linker.cc
       │
       ├─ DexFile::GetStringDataAndUtf16Length()       // 从 DEX 文件读取 UTF-8 数据
       │
       ├─ InternTable::InternStrong() 或 InternWeak() // runtime/intern_table.cc
       │    └─ 在字符串池中查找或创建 mirror::String 对象
       │
       └─ DexCache::SetResolvedString()               // 存入缓存供后续使用
```

## 关键文件

| 文件 | 职责 |
|------|------|
| `runtime/class_linker-inl.h` | ResolveString 内联实现，快速路径 |
| `runtime/class_linker.cc` | DoResolveString 慢速路径 |
| `runtime/class_linker.h` | 接口声明（3 个重载） |
| `runtime/mirror/dex_cache-inl.h` | DexCache 缓存读写 |
| `runtime/intern_table.cc` | 字符串池（intern table）管理 |
| `runtime/entrypoints/quick/quick_dexcache_entrypoints.cc` | 编译代码的运行时入口 |
| `runtime/interpreter/interpreter_switch_impl-inl.h` | 解释器 const-string 处理 |

## 编译器侧

各架构代码生成器在遇到 `HLoadString` 节点时，会生成对 `artResolveStringFromCode` 的调用：

- `compiler/optimizing/code_generator_arm64.cc`
- `compiler/optimizing/code_generator_arm_vixl.cc`
- `compiler/optimizing/code_generator_x86_64.cc`
- `compiler/optimizing/code_generator_x86.cc`
- `compiler/optimizing/code_generator_riscv64.cc`

## 备注

- `ResolveString` 有 3 个重载：接受 `ArtField*`、`ArtMethod*`、`Handle<DexCache>` 作为 referrer。
- 字符串解析后会缓存在 DexCache 中，后续访问走快速路径直接返回。
- `InternStrong` vs `InternWeak`：由 `com::android::art::flags::weak_const_string()` 标志控制。
