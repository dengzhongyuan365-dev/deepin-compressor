# pzip 高性能并行压缩技术文档

## 1. 概述

### 1.1 背景

deepin-compressor 原有的 ZIP 压缩功能基于 libzip 库实现，采用单线程串行压缩方式。在处理大量小文件或大容量数据时，压缩性能成为瓶颈。为了提升用户体验，我们开发了 pzip 高性能并行压缩工具，并将其集成到 deepin-compressor 中。

### 1.2 目标

- 大幅提升 ZIP 压缩性能（目标：提升 50% 以上）
- 保持与标准 ZIP 格式的完全兼容
- 无缝集成到现有 GUI 工作流程
- 支持 ARM 和 x86 架构

## 2. 技术方案对比

### 2.1 原方案：libzip

| 特性 | 说明 |
|------|------|
| 压缩模型 | 单线程串行压缩 |
| 压缩算法 | zlib (Deflate) |
| 压缩级别 | 默认 level 6 |
| 文件处理 | 逐个文件顺序压缩 |
| CPU 利用率 | 单核 100%，多核闲置 |

**性能瓶颈**：
- 单线程无法利用多核 CPU
- 高压缩级别增加计算开销
- 文件 I/O 与压缩串行执行

### 2.2 新方案：pzip

| 特性 | 说明 |
|------|------|
| 压缩模型 | 多线程并行压缩 |
| 压缩算法 | 自研 FastDeflate |
| 压缩级别 | 默认 level 1 (BestSpeed) |
| 文件处理 | 多文件并行压缩 |
| CPU 利用率 | 充分利用所有 CPU 核心 |

## 3. 为什么不改造 libzip 而选择开发 pzip

在项目初期，我们评估了两种技术路线：改造现有 libzip 实现多线程 vs 开发全新的 pzip。最终选择后者，原因如下：

### 3.1 libzip 架构限制

libzip 是一个成熟的第三方开源库，其核心架构存在以下并行化障碍：

#### 3.1.1 顺序写入设计

```c
// libzip 的典型使用模式
zip_t *archive = zip_open("test.zip", ZIP_CREATE, &err);
for (each file) {
    zip_source_t *source = zip_source_file(archive, filename, 0, 0);
    zip_file_add(archive, name, source, ZIP_FL_OVERWRITE);  // 必须顺序调用
}
zip_close(archive);  // 此时才真正压缩和写入
```

**问题**：
- `zip_file_add` 不是线程安全的
- `zip_close` 时才进行压缩，无法并行化
- 内部状态机设计为单线程顺序执行

#### 3.1.2 压缩回调机制

```c
// libzip 内部压缩流程
static zip_int64_t compress_callback(void *state, void *data, 
                                      zip_uint64_t len, 
                                      zip_source_cmd_t cmd) {
    // 单一回调，无法拆分到多线程
    switch (cmd) {
        case ZIP_SOURCE_READ:
            return read_and_compress(state, data, len);
        // ...
    }
}
```

**问题**：
- 压缩与 I/O 紧耦合在回调中
- 无法将压缩任务分发到多个线程
- 修改需要重写整个 source 机制

#### 3.1.3 全局状态依赖

```c
// libzip 内部结构
struct zip {
    zip_entry_t *entry;           // 当前处理的条目
    zip_uint64_t nentry;          // 条目数量
    struct zip_hash *names;       // 文件名哈希表（非线程安全）
    // ... 大量共享状态
};
```

**问题**：
- 多个共享数据结构需要加锁保护
- 加锁会严重影响并行效率
- 改造风险高，可能引入难以调试的并发 bug

### 3.2 ZIP 格式本身的限制

#### 3.2.1 文件头依赖

```
ZIP 文件结构:
┌──────────────────────┐
│ Local File Header 1  │ ◄── 需要知道压缩后大小（或用 Data Descriptor）
├──────────────────────┤
│ Compressed Data 1    │
├──────────────────────┤
│ Local File Header 2  │ ◄── 偏移量依赖前一个文件
├──────────────────────┤
│ Compressed Data 2    │
├──────────────────────┤
│ ...                  │
├──────────────────────┤
│ Central Directory    │ ◄── 需要所有文件的偏移量
├──────────────────────┤
│ End of Central Dir   │
└──────────────────────┘
```

**并行化难点**：
- 每个文件的偏移量依赖前面所有文件的压缩后大小
- Central Directory 需要所有文件信息
- 无法真正"同时写入"多个文件

#### 3.2.2 libzip 的预计算模式

```c
// libzip 在 zip_close 时的处理流程
1. 遍历所有待添加文件
2. 逐个读取、压缩、写入
3. 记录每个文件的偏移和大小
4. 写入 Central Directory
```

这种设计是为了：
- 确保文件完整性
- 支持原地更新已有 ZIP
- 简化错误处理

但也导致了无法并行化。

### 3.3 改造成本分析

| 改造项 | 工作量 | 风险 |
|--------|--------|------|
| 重写 zip_source 机制 | 2-3 周 | 高 |
| 添加线程安全锁 | 1 周 | 中 |
| 修改压缩回调 | 1-2 周 | 高 |
| 实现并行调度 | 1-2 周 | 中 |
| 测试与调试 | 2-4 周 | 高 |
| **总计** | **7-12 周** | **高** |

**对比开发 pzip**：

| 开发项 | 工作量 | 风险 |
|--------|--------|------|
| FastDeflate 压缩器 | 2 周 | 中 |
| 并行调度框架 | 1 周 | 低 |
| ZIP 写入器 | 1 周 | 低 |
| 集成与测试 | 1 周 | 低 |
| **总计** | **5 周** | **中** |

### 3.4 维护成本考量

#### 改造 libzip 的维护问题

1. **上游同步困难**
   - libzip 持续更新，我们的修改难以合并
   - 每次上游更新都需要重新适配
   - 可能错过重要的安全修复

2. **调试复杂度**
   - 并发 bug 难以复现和定位
   - 需要深入理解 libzip 内部实现
   - 文档和社区支持有限

3. **兼容性风险**
   - 修改可能破坏现有功能
   - 需要大量回归测试

#### pzip 的维护优势

1. **完全自主可控**
   - 代码量小（~2000 行），易于理解
   - 无外部依赖冲突
   - 可针对性优化

2. **清晰的并行设计**
   - 从零开始设计的并行架构
   - 无历史包袱
   - 易于扩展

### 3.5 技术决策总结

```
┌─────────────────────────────────────────────────────────────┐
│                    技术路线评估矩阵                          │
├─────────────────┬──────────────────┬───────────────────────┤
│      维度        │   改造 libzip    │      开发 pzip        │
├─────────────────┼──────────────────┼───────────────────────┤
│ 开发周期        │ 7-12 周          │ 5 周                  │
│ 技术风险        │ 高               │ 中                    │
│ 性能上限        │ 受限于架构       │ 可充分优化            │
│ 维护成本        │ 高（需跟踪上游） │ 低（自主可控）        │
│ 代码可读性      │ 差（大量补丁）   │ 好（全新设计）        │
│ 扩展性          │ 差               │ 好                    │
├─────────────────┼──────────────────┼───────────────────────┤
│ **最终选择**    │        ✗         │         ✓             │
└─────────────────┴──────────────────┴───────────────────────┘
```

**结论**：开发全新的 pzip 是更优的技术路线。虽然看起来"重复造轮子"，但考虑到 libzip 的架构限制、改造风险和维护成本，从零开发一个专门针对并行压缩优化的工具是更明智的选择。

## 4. 核心算法设计

### 3.1 FastDeflate 压缩算法

FastDeflate 是针对高速压缩场景优化的 Deflate 实现，提供两个压缩级别：

#### 3.1.1 FastEncL1 (Level 1 - BestSpeed)

```
设计目标：最快压缩速度，适合大量小文件
匹配算法：简单哈希链匹配
最小匹配长度：3 字节
最大匹配长度：258 字节
哈希表大小：64KB
查找深度：1 (仅检查第一个匹配)
```

**匹配流程**：
```
1. 计算当前位置 3 字节哈希值
2. 在哈希表中查找匹配位置
3. 如果匹配长度 >= 3，输出 (length, distance) 对
4. 否则输出字面字节
5. 更新哈希表
```

#### 3.1.2 FastEncL4 (Level 4+)

```
设计目标：平衡压缩率和速度
匹配算法：哈希链 + 延迟匹配
最小匹配长度：3 字节
最大匹配长度：258 字节
哈希表大小：64KB
查找深度：4-16 (根据级别调整)
```

**延迟匹配优化**：
```
当找到一个匹配后，检查下一个位置是否有更好的匹配
如果下一位置匹配更优，则输出当前字节为字面量
继续在下一位置寻找最佳匹配
```

### 3.2 Huffman 编码

采用动态 Huffman 编码，针对每个 64KB 块独立构建码表：

```cpp
struct HuffmanBitWriter {
    // 字面量/长度码表 (0-285)
    uint16_t litLenCode[286];
    uint8_t litLenBits[286];
    
    // 距离码表 (0-29)
    uint16_t distCode[30];
    uint8_t distBits[30];
    
    // 码长编码码表 (0-18)
    uint16_t codeLenCode[19];
    uint8_t codeLenBits[19];
};
```

**编码流程**：
```
1. 统计字面量和匹配的频率
2. 构建最优 Huffman 树
3. 生成码表并写入块头
4. 使用码表编码数据
5. 写入块结束标记 (EOB)
```

### 3.3 数据块结构

每个压缩块采用以下结构：

```
+----------------+
| Block Header   |  3 bits: BFINAL(1) + BTYPE(2)
+----------------+
| Huffman Trees  |  动态码表
+----------------+
| Compressed Data|  编码后的数据
+----------------+
| EOB Symbol     |  块结束标记 (256)
+----------------+
```

## 5. 并行架构设计

### 4.1 整体架构

```
┌─────────────────────────────────────────────────────────┐
│                    Main Thread                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │
│  │ File Scanner│──│ Task Queue  │──│ ZIP Writer  │     │
│  └─────────────┘  └──────┬──────┘  └─────────────┘     │
│                          │                              │
│         ┌────────────────┼────────────────┐            │
│         ▼                ▼                ▼            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐    │
│  │  Worker 1   │  │  Worker 2   │  │  Worker N   │    │
│  │ FastDeflate │  │ FastDeflate │  │ FastDeflate │    │
│  └─────────────┘  └─────────────┘  └─────────────┘    │
└─────────────────────────────────────────────────────────┘
```

### 4.2 WorkerPool 实现

```cpp
class WorkerPool {
    std::vector<std::thread> workers_;
    std::queue<std::function<void()>> tasks_;
    std::mutex mutex_;
    std::condition_variable cv_;
    bool shutdown_ = false;
    
public:
    WorkerPool(size_t threads = 0) {
        if (threads == 0) {
            threads = std::thread::hardware_concurrency();
        }
        for (size_t i = 0; i < threads; ++i) {
            workers_.emplace_back([this] { workerLoop(); });
        }
    }
    
    template<typename F>
    auto submit(F&& f) -> std::future<decltype(f())>;
};
```

### 4.3 文件压缩流水线

```
阶段 1: 文件扫描
  └── 递归遍历目录
  └── 收集文件元信息
  └── 计算总大小

阶段 2: 并行压缩
  └── 每个文件独立压缩
  └── 生成压缩数据块
  └── 计算 CRC32

阶段 3: ZIP 写入
  └── 写入 Local File Header
  └── 写入压缩数据
  └── 记录 Central Directory
  └── 写入 End of Central Directory
```

## 6. ZIP 格式实现

### 5.1 文件结构

```
ZIP Archive:
├── Local File Header 1
│   └── Compressed Data 1
├── Local File Header 2
│   └── Compressed Data 2
├── ...
├── Central Directory Header 1
├── Central Directory Header 2
├── ...
└── End of Central Directory Record
```

### 5.2 关键数据结构

```cpp
// Local File Header (30 bytes + variable)
struct LocalFileHeader {
    uint32_t signature = 0x04034b50;
    uint16_t version = 20;
    uint16_t flags = 0x0008;  // Data Descriptor
    uint16_t compression = 8; // Deflate
    uint16_t modTime;
    uint16_t modDate;
    uint32_t crc32;
    uint32_t compressedSize;
    uint32_t uncompressedSize;
    uint16_t nameLength;
    uint16_t extraLength;
};

// Data Descriptor (16 bytes)
struct DataDescriptor {
    uint32_t signature = 0x08074b50;
    uint32_t crc32;
    uint32_t compressedSize;
    uint32_t uncompressedSize;
};
```

### 5.3 流式写入

由于采用并行压缩，无法预知压缩后大小，因此使用 Data Descriptor：

```
1. 写入 Local File Header (sizes = 0)
2. 压缩并写入数据
3. 写入 Data Descriptor (包含实际大小和 CRC)
```

## 7. 性能优化

### 6.1 内存优化

```cpp
// 使用 thread_local 避免频繁分配
thread_local std::vector<uint8_t> compressBuffer;
thread_local FastDeflate deflater;

// 预分配缓冲区
compressBuffer.reserve(BLOCK_SIZE + BLOCK_SIZE / 8);
```

### 6.2 I/O 优化

```cpp
// 大缓冲区减少系统调用
static constexpr size_t WRITE_BUFFER_SIZE = 256 * 1024;
char writeBuffer[WRITE_BUFFER_SIZE];
outputFile.rdbuf()->pubsetbuf(writeBuffer, sizeof(writeBuffer));
```

### 6.3 查找表优化

```cpp
// 编译时计算的查找表
alignas(64) static constexpr uint8_t lengthExtraBits[29] = {...};
alignas(64) static constexpr uint16_t lengthBase[29] = {...};
alignas(64) static constexpr uint8_t distExtraBits[30] = {...};
alignas(64) static constexpr uint16_t distBase[30] = {...};
```

## 8. 集成方案

### 7.1 插件架构

```
deepin-compressor/
├── 3rdparty/
│   ├── pzip/                    # pzip 核心库
│   │   ├── include/pzip/
│   │   │   ├── archiver.h
│   │   │   ├── fast_deflate.h
│   │   │   └── ...
│   │   ├── src/
│   │   │   ├── archiver.cpp
│   │   │   ├── fast_deflate.cpp
│   │   │   └── ...
│   │   └── CMakeLists.txt
│   │
│   └── clipzipplugin/           # 插件封装
│       ├── clipzipplugin.h
│       ├── clipzipplugin.cpp
│       ├── kerfuffle_clipzip.json
│       └── CMakeLists.txt
```

### 7.2 插件配置

```json
{
    "KPlugin": {
        "Id": "kerfuffle_clipzip",
        "MimeTypes": ["application/zip"],
        "Name": "pzip plugin"
    },
    "X-KDE-Priority": 200,
    "X-KDE-Kerfuffle-ReadWrite": true,
    "application/zip": {
        "CompressionLevelDefault": 1,
        "CompressionLevelMax": 9,
        "CompressionLevelMin": 1
    }
}
```

### 7.3 条件编译

```cmake
# 仅 x86/ARM + Qt6 环境启用
if((CMAKE_SYSTEM_PROCESSOR MATCHES "aarch64|arm|x86_64|amd64") 
   AND (QT_VERSION_MAJOR EQUAL 6))
    add_subdirectory(pzip)
    add_subdirectory(clipzipplugin)
endif()
```

### 7.4 插件选择策略

```cpp
// uitools.cpp
// 压缩时：优先使用 pzip (priority=200 > libzip=170)
// 解压时：跳过 pzip，使用 libzip
bool removePzipFlag = (!bWrite) && (mimeType == "application/zip");
if (removePzipFlag && plugin->name().contains("pzip")) {
    continue;  // 跳过 pzip
}
```

## 9. 性能测试

### 8.1 测试环境

| 项目 | ARM 环境 | x86 环境 |
|------|----------|----------|
| CPU | 鲲鹏 920 | Intel Core |
| 核心数 | 8 | 8 |
| 内存 | 16GB | 16GB |
| 存储 | SSD | SSD |
| 系统 | UOS V20 | UOS V20 |
| Qt | Qt6 | Qt6 |

### 8.2 测试数据

- 测试集：2.5GB 碎文件（约 300+ 个文件）
- 文件类型：文档、图片、视频、安装包等混合

### 8.3 测试结果

| 方案 | ARM 耗时 | x86 耗时 | 压缩率 |
|------|----------|----------|--------|
| libzip (level 6) | ~12s | ~10s | ~65% |
| pzip (level 1) | ~3.5s | ~5s | ~60% |
| **提升** | **71%** | **50%** | -5% |

### 8.4 CPU 利用率

```
libzip:  [████████░░░░░░░░] 1 核 100%
pzip:    [████████████████] 8 核 100%
```

## 10. 已知限制

1. **仅支持压缩**：pzip 插件仅用于压缩，解压由 libzip 处理
2. **不支持加密**：当前版本不支持 ZIP 加密
3. **不支持追加**：不支持向已有 ZIP 追加文件
4. **压缩率略低**：level 1 压缩率比 level 6 低约 5%

## 11. 后续优化方向

1. **自适应压缩级别**：根据文件类型自动选择压缩级别
2. **增量压缩**：支持向已有 ZIP 追加文件
3. **内存映射**：使用 mmap 优化大文件读取
4. **SIMD 加速**：使用 NEON/SSE 加速 CRC32 和匹配查找

## 12. 参考资料

- [RFC 1951 - DEFLATE Compressed Data Format](https://tools.ietf.org/html/rfc1951)
- [ZIP File Format Specification](https://pkware.cachefly.net/webdocs/casestudies/APPNOTE.TXT)
- [zlib Technical Details](https://www.zlib.net/zlib_tech.html)

---

文档版本：1.0  
更新日期：2026-01-15  
作者：deepin-compressor 开发团队

