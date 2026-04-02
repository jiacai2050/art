# DEX 文件格式源码分析 —— 基于 ART 源码

本文结合 `art` 源码，分析 DEX 文件各模块的数据结构定义、加载流程、校验机制，以及它们在运行时和编译时如何协调工作。

---

## 1. 源码模块总览

```
mi-art/
├── libdexfile/dex/          # DEX 文件格式核心定义与解析
│   ├── dex_file.h/cc        # DexFile 基类：Header、常量池访问、数据区访问
│   ├── dex_file_structs.h   # 原始结构体：StringId, TypeId, FieldId, MethodId, ClassDef...
│   ├── dex_file_types.h     # 索引类型：StringIndex, TypeIndex, ProtoIndex
│   ├── dex_file_loader.h/cc # 加载器：从 APK/文件/内存加载 DEX
│   ├── dex_file_verifier.h/cc # 格式校验器
│   ├── standard_dex_file.h  # 标准 DEX 格式（APK 中的原始格式）
│   ├── compact_dex_file.h   # CompactDex 格式（ART 内部优化格式）
│   ├── class_accessor.h     # class_data_item 的高层访问接口
│   ├── code_item_accessors.h # code_item 的统一访问接口
│   ├── dex_instruction.h    # Dalvik 字节码指令定义
│   ├── type_lookup_table.h  # 类型快速查找哈希表
│   └── dex_file_layout.h    # 布局优化（hot/cold 分区）
├── runtime/
│   ├── class_linker.h/cc    # 类链接器：RegisterDexFile, 类加载与解析
│   ├── oat/oat_file_manager.h # OAT 文件管理：OpenDexFilesFromOat
│   └── jit/                 # JIT 编译器
├── dex2oat/
│   ├── dex2oat.cc           # AOT 编译入口
│   └── linker/oat_writer.h/cc # OAT 文件写入
└── compiler/                # 优化编译器
```

---

## 2. DEX Header 结构（源码定义）

来自 `libdexfile/dex/dex_file.h` 中 `DexFile::Header`：

```cpp
// libdexfile/dex/dex_file.h (line ~140)
struct Header {
    Magic magic_ = {};                  // "dex\n035\0" (8 bytes)
    uint32_t checksum_ = 0;            // Adler-32 校验（跳过 magic + checksum 本身）
    Sha1 signature_ = {};              // SHA-1 哈希（20 bytes，跳过 magic/checksum/signature）
    uint32_t file_size_ = 0;           // 整个 DEX 文件大小
    uint32_t header_size_ = 0;         // Header 大小（标准 0x70）
    uint32_t endian_tag_ = 0;          // 0x12345678 = 小端
    uint32_t link_size_ = 0;           // 链接段大小（通常为 0）
    uint32_t link_off_ = 0;            // 链接段偏移
    uint32_t map_off_ = 0;             // MapList 偏移（相对 data_off_）
    uint32_t string_ids_size_ = 0;     // StringId 数量
    uint32_t string_ids_off_ = 0;      // StringId 数组偏移
    uint32_t type_ids_size_ = 0;       // TypeId 数量（上限 65535）
    uint32_t type_ids_off_ = 0;
    uint32_t proto_ids_size_ = 0;      // ProtoId 数量（上限 65535）
    uint32_t proto_ids_off_ = 0;
    uint32_t field_ids_size_ = 0;      // FieldId 数量
    uint32_t field_ids_off_ = 0;
    uint32_t method_ids_size_ = 0;     // MethodId 数量
    uint32_t method_ids_off_ = 0;
    uint32_t class_defs_size_ = 0;     // ClassDef 数量
    uint32_t class_defs_off_ = 0;
    uint32_t data_size_ = 0;           // 数据区大小
    uint32_t data_off_ = 0;            // 数据区偏移
};

// DEX V41 扩展 Header，支持 Container
struct HeaderV41 : public Header {
    uint32_t container_size_ = 0;      // 容器中所有 DEX 的总大小
    uint32_t header_offset_ = 0;       // 本 DEX header 在容器中的偏移
};
```

关键常量：

```cpp
static constexpr size_t kDexMagicSize = 4;          // "dex\n"
static constexpr size_t kDexVersionLen = 4;          // "035\0"
static constexpr uint32_t kDexContainerVersion = 41; // V41 引入 container
static constexpr uint32_t kDexEndianConstant = 0x12345678;
static constexpr size_t kSha1DigestSize = 20;
```

---

## 3. 常量池结构体（ID 表）

来自 `libdexfile/dex/dex_file_structs.h`：

```cpp
// 字符串标识符 —— 指向 data 区的 MUTF-8 字符串
struct StringId {
    uint32_t string_data_off_;  // 偏移到 string_data_item
};

// 类型标识符 —— 通过 StringIndex 间接引用类型描述符字符串
struct TypeId {
    dex::StringIndex descriptor_idx_;  // 索引到 string_ids
};

// 方法原型 —— 描述方法签名（返回类型 + 参数列表）
struct ProtoId {
    dex::StringIndex shorty_idx_;      // 短描述符，如 "VIL"
    dex::TypeIndex return_type_idx_;   // 返回类型
    uint16_t pad_;                     // 对齐填充
    uint32_t parameters_off_;          // 指向 TypeList（参数类型列表）
};

// 字段标识符
struct FieldId {
    dex::TypeIndex class_idx_;         // 所属类
    dex::TypeIndex type_idx_;          // 字段类型
    dex::StringIndex name_idx_;        // 字段名
};

// 方法标识符
struct MethodId {
    dex::TypeIndex class_idx_;         // 所属类
    dex::ProtoIndex proto_idx_;        // 方法原型
    dex::StringIndex name_idx_;        // 方法名
};
```

索引类型定义在 `dex_file_types.h` 中，使用模板实现类型安全：

```cpp
// libdexfile/dex/dex_file_types.h
class StringIndex : public DexIndex<uint32_t> { ... };  // 32-bit
class TypeIndex   : public DexIndex<uint16_t> { ... };  // 16-bit（上限 65535）
class ProtoIndex  : public DexIndex<uint16_t> { ... };  // 16-bit
```

**间接引用链示例**：查找方法 `Foo.bar(int)V`

```
MethodId
  ├── class_idx_  → TypeId[i].descriptor_idx_ → StringId[j] → "LFoo;"
  ├── name_idx_   → StringId[k] → "bar"
  └── proto_idx_  → ProtoId[m]
                      ├── shorty_idx_ → StringId[n] → "VI"
                      ├── return_type_idx_ → TypeId[p] → "V"
                      └── parameters_off_ → TypeList → [TypeId[q] → "I"]
```

---

## 4. 类定义与类数据

```cpp
// libdexfile/dex/dex_file_structs.h
struct ClassDef {
    dex::TypeIndex class_idx_;         // 类类型索引
    uint16_t pad1_;
    uint32_t access_flags_;            // public/final/abstract/interface 等
    dex::TypeIndex superclass_idx_;    // 父类类型索引
    uint16_t pad2_;
    uint32_t interfaces_off_;          // → TypeList（实现的接口列表）
    dex::StringIndex source_file_idx_; // 源文件名
    uint32_t annotations_off_;         // → AnnotationsDirectoryItem
    uint32_t class_data_off_;          // → class_data_item（字段和方法列表）
    uint32_t static_values_off_;       // → EncodedArray（静态字段初始值）
};
```

`class_data_item` 通过 `ClassAccessor`（`class_accessor.h`）访问，使用 LEB128 差值编码：

```cpp
// libdexfile/dex/class_accessor.h
class ClassAccessor {
  class Method : public BaseItem {
    uint32_t GetCodeItemOffset() const;     // 获取方法字节码偏移
    InvokeType GetInvokeType(...) const;    // direct/virtual/static/interface
    uint32_t GetIndex() const;              // method_idx（差值累加后的绝对索引）
    uint32_t GetAccessFlags() const;        // 访问标志
  };

  class Field : public BaseItem {
    uint32_t GetIndex() const;              // field_idx
    uint32_t GetAccessFlags() const;
  };

  // 遍历所有字段和方法
  void VisitFieldsAndMethods(...);
};
```

---

## 5. 字节码结构（CodeItem）

标准 DEX 的 CodeItem 定义在 `standard_dex_file.h`：

```cpp
// libdexfile/dex/standard_dex_file.h
struct CodeItem : public dex::CodeItem {
    uint16_t registers_size_;              // 寄存器总数（locals + params）
    uint16_t ins_size_;                    // 入参占用的寄存器数
    uint16_t outs_size_;                   // 调用其他方法时需要的寄存器数
    uint16_t tries_size_;                  // try-catch 块数量
    uint32_t debug_info_off_;              // 调试信息偏移
    uint32_t insns_size_in_code_units_;    // 指令数组大小（以 2 字节为单位）
    uint16_t insns_[1];                    // 实际字节码数组
    // 如果 tries_size_ > 0，后面紧跟 padding + TryItem[] + catch handler 数据
};
```

CompactDex 的 CodeItem 更紧凑（`compact_dex_file.h`），将 registers/ins/outs/tries 压缩到一个 `uint16_t fields_` 中（每个 4 bit），大方法通过 preheader 扩展：

```cpp
// libdexfile/dex/compact_dex_file.h
struct CodeItem : public dex::CodeItem {
    // fields_ 低 16 位编码: [registers:4][ins:4][outs:4][tries:4]
    // insns_count_and_flags_ 编码指令数量和 preheader 标志
    // 99% 的方法不需要 preheader，实现快速路径
    uint16_t fields_;
    uint16_t insns_count_and_flags_;
    uint16_t insns_[];
};
```

统一访问接口 `CodeItemDataAccessor`（`code_item_accessors.h`）屏蔽两种格式差异：

```cpp
// libdexfile/dex/code_item_accessors.h
class CodeItemDataAccessor : public CodeItemInstructionAccessor {
    uint16_t RegistersSize() const;
    uint16_t InsSize() const;
    uint16_t OutsSize() const;
    uint16_t TriesSize() const;
    const uint16_t* Insns() const;
    uint32_t InsnsSizeInCodeUnits() const;
    // 迭代 try-catch 块
    IterationRange<const dex::TryItem*> TryItems() const;
};
```

---

## 6. MapList —— 文件目录

```cpp
// libdexfile/dex/dex_file_structs.h
struct MapItem {
    uint16_t type_;    // MapItemType 枚举值
    uint16_t unused_;
    uint32_t size_;    // 该类型的条目数量
    uint32_t offset_;  // 在文件中的偏移
};

struct MapList {
    uint32_t size_;        // MapItem 数量
    MapItem list_[1];      // 变长数组
};
```

MapItemType 枚举（`dex_file.h`）：

```cpp
enum MapItemType : uint16_t {
    kDexTypeHeaderItem               = 0x0000,
    kDexTypeStringIdItem             = 0x0001,
    kDexTypeTypeIdItem               = 0x0002,
    kDexTypeProtoIdItem              = 0x0003,
    kDexTypeFieldIdItem              = 0x0004,
    kDexTypeMethodIdItem             = 0x0005,
    kDexTypeClassDefItem             = 0x0006,
    kDexTypeCallSiteIdItem           = 0x0007,
    kDexTypeMethodHandleItem         = 0x0008,
    kDexTypeMapList                  = 0x1000,
    kDexTypeTypeList                 = 0x1001,
    kDexTypeAnnotationSetRefList     = 0x1002,
    kDexTypeAnnotationSetItem        = 0x1003,
    kDexTypeClassDataItem            = 0x2000,
    kDexTypeCodeItem                 = 0x2001,
    kDexTypeStringDataItem           = 0x2002,
    kDexTypeDebugInfoItem            = 0x2003,
    kDexTypeAnnotationItem           = 0x2004,
    kDexTypeEncodedArrayItem         = 0x2005,
    kDexTypeAnnotationsDirectoryItem = 0x2006,
    kDexTypeHiddenapiClassData       = 0xF000,  // Android 特有：隐藏 API 数据
};
```

---

## 7. 模块协作流程

### 7.1 DEX 加载流程（DexFileLoader）

```
APK/文件/内存
      │
      ▼
DexFileLoader::Open()          ← libdexfile/dex/dex_file_loader.cc
      │
      ├─ 读取 magic，判断是 ZIP 还是裸 DEX
      │   ├─ ZIP: 解压 classes.dex, classes2.dex, ...（MultiDex）
      │   └─ 裸 DEX: 直接 mmap
      │
      ├─ 根据 magic 区分 StandardDexFile vs CompactDexFile
      │   ├─ "dex\n" → StandardDexFile
      │   └─ "cdex"  → CompactDexFile
      │
      ├─ DexFileContainer 管理物理存储
      │   ├─ MemoryDexFileContainer（内存缓冲区）
      │   ├─ MemMapContainer（mmap 映射）
      │   └─ VectorContainer（std::vector 拥有的数据）
      │
      └─ dex::Verify()          ← libdexfile/dex/dex_file_verifier.cc
          │  校验 header、各 ID 表的偏移/大小/排序/引用合法性
          └─ 返回 DexFile 对象
```

源码关键路径（`dex_file_loader.cc`）：

```cpp
// 判断 magic 类型
bool DexFileLoader::IsMagicValid(const uint8_t* magic) {
    return StandardDexFile::IsMagicValid(magic);
}

// MultiDex 命名规则
std::string DexFileLoader::GetMultiDexClassesDexName(size_t index) {
    return (index == 0) ? "classes.dex" : StringPrintf("classes%zu.dex", index + 1);
}

// MultiDex 位置格式: "base.apk!classes2.dex"
static constexpr char kMultiDexSeparator = '!';
```

### 7.2 DEX 校验流程（DexFileVerifier）

校验器（`dex_file_verifier.cc`）使用 bitmap 追踪已验证的 section：

```cpp
// 每种 MapItemType 映射到一个 bit
constexpr uint32_t MapTypeToBitMask(DexFile::MapItemType map_item_type) {
    switch (map_item_type) {
        case DexFile::kDexTypeHeaderItem:    return 1 << 0;
        case DexFile::kDexTypeStringIdItem:  return 1 << 1;
        case DexFile::kDexTypeTypeIdItem:    return 1 << 2;
        // ... 每种类型一个 bit，确保不重复、不遗漏
    }
}
```

校验内容包括：
- Header 字段合法性（magic、版本、大小、对齐）
- 各 ID 表的偏移在文件范围内
- 索引引用不越界（如 TypeId.descriptor_idx_ < string_ids_size_）
- 字符串排序（string_ids 按 MUTF-8 字典序）
- ClassDef 排序（V37+ 强制按 type_idx 排序）
- CodeItem 的寄存器数、指令合法性

### 7.3 运行时类加载（ClassLinker ↔ DexFile）

```
OatFileManager::OpenDexFilesFromOat()     ← runtime/oat/oat_file_manager.h
      │
      ├─ 尝试找到已编译的 OAT 文件
      │   └─ OatDexFile::OpenDexFile()    ← 从 OAT 中提取 DEX
      │
      ├─ 如果没有 OAT，回退到 DexFileLoader 加载原始 DEX
      │
      └─ ClassLinker::RegisterDexFile()   ← runtime/class_linker.h
            │
            ├─ 创建 DexCache（缓存已解析的字符串、类型、方法等）
            │
            └─ 后续类加载时：
                ClassLinker::FindClass()
                  ├─ TypeLookupTable::Lookup()  ← 哈希表快速查找 class_def_idx
                  ├─ DexFile::GetClassDef()     ← 获取 ClassDef
                  ├─ ClassAccessor 遍历字段和方法
                  └─ CodeItemDataAccessor 读取字节码
```

`RegisterDexFile` 的签名：

```cpp
// runtime/class_linker.h (line ~502)
ObjPtr<mirror::DexCache> RegisterDexFile(
    const DexFile& dex_file,
    ObjPtr<mirror::ClassLoader> class_loader)
    REQUIRES(!Locks::dex_lock_)
    REQUIRES_SHARED(Locks::mutator_lock_);
```

### 7.4 AOT 编译流程（dex2oat）

```
dex2oat::Setup()                          ← dex2oat/dex2oat.cc (line ~1479)
      │
      ├─ CreateOatWriters()               创建 OAT 写入器
      ├─ AddDexFileSources()              添加 DEX 文件源
      ├─ OatWriter::WriteAndOpenDexFiles() 将 DEX 写入 VDEX/OAT
      │     └─ DexFileLoader 加载并验证每个 DEX
      │
      ├─ CompilerDriver 编译每个方法
      │     ├─ ClassAccessor 遍历 class_data_item
      │     ├─ CodeItemDataAccessor 读取字节码
      │     └─ 优化编译器生成机器码
      │
      └─ OatWriter 写出最终 OAT 文件
            ├─ DEX 数据（可能是 CompactDex）
            ├─ 编译后的机器码
            └─ 各种元数据（GC map、stack map 等）
```

核心代码（`dex2oat.cc`）：

```cpp
// dex2oat/dex2oat.cc (line ~1516)
if (!oat_writers_[i]->WriteAndOpenDexFiles(
    vdex_files_[i].get(),
    verify,                    // 是否需要验证（有 vdex 时可跳过）
    use_existing_vdex_,
    copy_dex_files_,
    &opened_dex_files_map,
    &opened_dex_files)) {
  return dex2oat::ReturnCode::kOther;
}
// 建立 dex_file → oat_index 映射
dex_file_oat_index_map_.insert(std::make_pair(dex_file.get(), i));
```

---

## 8. StandardDexFile vs CompactDexFile

| 特性 | StandardDexFile | CompactDexFile |
|------|----------------|----------------|
| Magic | `dex\n` | `cdex` |
| 来源 | APK 中原始格式 | ART 内部生成（VDEX/OAT） |
| CodeItem | 固定布局，每字段独立 | 压缩布局，4-bit 编码 + preheader |
| debug_info | CodeItem 内 `debug_info_off_` | 独立的 compact offset table |
| 数据共享 | 无 | 多个 DEX 可共享 data section |
| 对齐 | CodeItem 4 字节对齐 | CodeItem 2 字节对齐 |

两者通过 `DexFile` 基类的虚函数和 `CodeItemDataAccessor` 统一访问：

```cpp
// dex_file.h
class DexFile {
    bool IsCompactDexFile() const { return is_compact_dex_; }
    bool IsStandardDexFile() const { return !is_compact_dex_; }
    const StandardDexFile* AsStandardDexFile() const;
    const CompactDexFile* AsCompactDexFile() const;
    virtual uint32_t GetCodeItemSize(const dex::CodeItem&) const = 0;
};
```

---

## 9. 布局优化（DexLayout）

`dex_file_layout.h` 定义了基于 profile 的布局优化策略：

```cpp
enum class LayoutType : uint8_t {
    kLayoutTypeHot,            // 热代码，应 pin 到内存
    kLayoutTypeSometimesUsed,  // 随机访问
    kLayoutTypeStartupOnly,    // 仅启动时使用，之后可 madvise
    kLayoutTypeUsedOnce,       // 仅用一次（如 <clinit>）
    kLayoutTypeUnused,         // 未使用
};

class DexLayoutSection {
    class Subsection {
        uint32_t start_offset_;
        uint32_t end_offset_;
    };
    Subsection parts_[kLayoutTypeCount];  // 每种布局类型一个子段
};

class DexLayoutSections {
    enum class SectionType : uint8_t {
        kSectionTypeCode,      // 代码段布局
        kSectionTypeStrings,   // 字符串段布局
    };
    DexLayoutSection sections_[kSectionCount];
};
```

这使得 ART 可以根据运行时 profile 将热代码和冷代码分开排列，优化内存页的使用。

---

## 10. TypeLookupTable —— 类查找加速

```cpp
// libdexfile/dex/type_lookup_table.h
class TypeLookupTable {
    // 编译时创建，写入 OAT 文件
    static TypeLookupTable Create(const DexFile& dex_file);

    // 运行时从 mmap 数据打开（零拷贝）
    static TypeLookupTable Open(const uint8_t* dex_data_pointer,
                                const uint8_t* raw_data,
                                uint32_t num_class_defs);

    // 通过类描述符 + hash 快速查找 class_def_idx
    // 返回 dex::kDexNoIndex 表示未找到
    uint32_t Lookup(std::string_view str, uint32_t hash) const;
};
```

工作流程：
1. `dex2oat` 编译时调用 `TypeLookupTable::Create()` 为每个 DEX 构建哈希表
2. 哈希表写入 OAT 文件
3. 运行时 `TypeLookupTable::Open()` 直接映射内存，无需重建
4. `ClassLinker::FindClass()` 调用 `Lookup()` 实现 O(1) 类查找

---

## 11. 端到端示例：从 Java 源码到运行时执行

```java
// Foo.java
public class Foo {
    public int add(int a, int b) { return a + b; }
}
```

### 编译阶段

1. `javac` → `Foo.class`
2. `d8/dx` → `classes.dex`，其中：
   - `string_ids`: `["Foo", "Ljava/lang/Object;", "LFoo;", "add", "III", ...]`
   - `type_ids`: `[→"LFoo;", →"Ljava/lang/Object;", →"I", ...]`
   - `proto_ids`: `[{shorty→"III", return→I, params→[I,I]}]`
   - `method_ids`: `[{class→Foo, name→"add", proto→上述}]`
   - `class_defs`: `[{class→Foo, super→Object, class_data→...}]`
   - `code_item`: `{regs=4, ins=3, outs=0, insns=[add-int v0,v2,v3; return v0]}`

### dex2oat 阶段

3. `dex2oat` 读取 `classes.dex`：
   ```
   DexFileLoader::Open("base.apk")
     → 解压 classes.dex
     → dex::Verify() 校验格式
     → 返回 StandardDexFile 对象
   ```

4. 编译 `Foo.add()`：
   ```
   ClassAccessor(dex_file, class_def)
     → 遍历 methods
     → CodeItemDataAccessor 读取 code_item
     → 优化编译器生成 ARM64 机器码
   ```

5. 输出 OAT 文件（含 TypeLookupTable）

### 运行时阶段

6. App 启动，加载 OAT：
   ```
   OatFileManager::OpenDexFilesFromOat("base.apk")
     → 找到匹配的 OAT 文件
     → ClassLinker::RegisterDexFile() 注册 DEX，创建 DexCache
   ```

7. 首次使用 `Foo` 类：
   ```
   ClassLinker::FindClass("LFoo;")
     → TypeLookupTable::Lookup("LFoo;", hash) → class_def_idx = 0
     → DexFile::GetClassDef(0) → ClassDef
     → 解析字段、方法、父类、接口
     → 链接完成，类可用
   ```

8. 调用 `Foo.add()`：
   - 如果有 AOT 编译的机器码 → 直接执行
   - 否则 → 解释执行 DEX 字节码 / JIT 编译

---

## 12. 总结：模块协作关系图

```
┌─────────────────────────────────────────────────────────────┐
│                        APK / DEX 文件                        │
└──────────────────────────┬──────────────────────────────────┘
                           │
                    DexFileLoader
                    (加载 + 解压)
                           │
              ┌────────────┼────────────┐
              ▼            ▼            ▼
        StandardDexFile  CompactDexFile  DexFileVerifier
              │            │            (格式校验)
              └─────┬──────┘
                    │
            DexFile 基类 API
         (Header, ID 表, DataPointer)
                    │
        ┌───────────┼───────────┐
        ▼           ▼           ▼
  ClassAccessor  CodeItem    TypeLookupTable
  (类数据遍历)  Accessor    (类查找加速)
                (字节码访问)
        │           │           │
        └───────────┼───────────┘
                    │
     ┌──────────────┼──────────────┐
     ▼              ▼              ▼
 ClassLinker     dex2oat        JIT
 (运行时类加载)  (AOT 编译)    (即时编译)
     │              │              │
     └──────────────┼──────────────┘
                    ▼
              OAT/VDEX 文件
         (编译产物 + DEX 数据)
```
