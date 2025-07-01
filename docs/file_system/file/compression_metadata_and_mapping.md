
# Indexlib 存储层压缩模块：元数据与地址映射深度解析

**涉及文件:**
*   `file_system/file/CompressFileInfo.h`
*   `file_system/file/CompressFileMeta.h`
*   `file_system/file/CompressFileAddressMapper.h`
*   `file_system/file/CompressFileAddressMapper.cpp`
*   `file_system/file/CompressFileAddressMapperEncoder.h`
*   `file_system/file/CompressFileAddressMapperEncoder.cpp`

## 1. 系统设计概览

压缩文件的核心挑战在于如何在保持高压缩率的同时，提供高效的随机访问（Random Access）能力。如果将整个文件视为一个连续的压缩流，那么读取其中任意一小段数据都需要从头解压，这在大多数场景下是不可接受的。Indexlib 的压缩框架通过**分块压缩**和**地址映射**机制完美地解决了这一问题。

本篇文档将深入探讨这一机制的核心——元数据管理与地址映射。其设计目标如下：

*   **高效查找**: 能够根据任意逻辑偏移（uncompressed offset）快速定位到其所在的压缩块（compressed block）及其在物理文件中的位置和大小。
*   **空间优化**: 地址映射信息本身也需要存储，必须对其进行有效压缩，以减少元数据带来的额外空间开销。
*   **信息完备**: 元数据需要包含所有重建压缩读取器所需的信息，如压缩算法、块大小、块数量、文件总长度等。
*   **扩展性**: 支持不同的元数据布局，例如将元数据和地址映射信息追加在数据文件末尾，或存储在独立的 `.meta` 文件中，以适应不同的文件系统和访问模式。

### 1.1. 架构与数据流

地址映射系统在写入和读取路径中扮演着不同的角色。

**写入路径**:
1.  数据由 `CompressDataDumper` 分块压缩。
2.  每当一个块被压缩后，其压缩后的大小被记录下来。
3.  `CompressFileAddressMapper` 在内存中维护一个动态增长的偏移量数组（`_blockOffsetVec`），记录每个压缩块的累积结束位置。
4.  当文件写入完成时，`CompressFileAddressMapper::Dump` 方法被调用，将内存中的偏移量数组进行编码（可选压缩），并写入文件。
5.  最后，`CompressFileInfo` 结构被填充并序列化为 JSON 字符串，写入 `.info` 文件。

**读取路径**:
1.  `IDirectory` 首先读取 `.info` 文件，反序列化得到 `CompressFileInfo` 对象。
2.  `CompressFileReaderCreator` 根据 `CompressFileInfo` 和文件类型创建合适的 `CompressFileReader`。
3.  在 `CompressFileReader::Init` 过程中，会创建 `CompressFileAddressMapper` 实例。
4.  `CompressFileAddressMapper::InitForRead` 被调用，它根据 `CompressFileInfo` 中的信息，从数据文件末尾或独立的 `.meta` 文件中读取地址映射数据，并构建起内存中的查找结构。
5.  当 `CompressFileReader::Read` 需要加载一个新块时，它会调用 `_compressAddrMapper` 的接口（如 `OffsetToBlockIdx`, `CompressBlockAddress`, `CompressBlockLength`）来完成逻辑偏移到物理块信息的翻译。

```mermaid
graph TD
    subgraph "写入时 (Write Path)"
        A[CompressDataDumper] -- "AddOneBlock(compSize)" --> B(CompressFileAddressMapper In-Memory)
        B -- "维护 _blockOffsetVec" --> C[Vector<size_t>]
        D[Close] -- "触发" --> E{Dump}
        E -- "编码/压缩" --> F(CompressFileAddressMapperEncoder)
        F -- "写入" --> G[物理文件末尾 或 .meta 文件]
        D -- "生成" --> H(CompressFileInfo)
        H -- "写入" --> I[.info 文件]
    end

    subgraph "读取时 (Read Path)"
        J[IDirectory] -- "读取" --> I
        I -- "反序列化" --> K(CompressFileInfo In-Memory)
        L[CompressFileReader] -- "创建" --> M{CompressFileAddressMapper For-Read}
        M -- "InitForRead" --> G
        G -- "加载" --> N[地址映射数据 (内存视图)]
        L -- "Read(offset)" --> O{地址翻译}
        O -- "OffsetToBlockIdx" --> M
        O -- "CompressBlockAddress" --> M
        O -- "CompressBlockLength" --> M
    end

    classDef write fill:#fee,stroke:#333,stroke-width:1px;
    classDef read fill:#eef,stroke:#333,stroke-width:1px;
    class A,B,C,D,E,F,H write;
    class J,K,L,M,N,O read;
```

## 2. 关键数据结构深度分析

### 2.1. `CompressFileInfo`: 顶层元数据

`CompressFileInfo` 是一个可被序列化为 JSON 的结构体，它存储在与压缩文件同名的 `.info` 文件中。它是重建压缩读取器的入口点，包含了所有必要的顶层信息。

```cpp
// file_system/file/CompressFileInfo.h

class CompressFileInfo : public autil::legacy::Jsonizable
{
public:
    // ...
    void Jsonize(autil::legacy::Jsonizable::JsonWrapper& json) override
    {
        json.Jsonize("compressor", compressorName, compressorName);
        json.Jsonize("block_count", blockCount, blockCount);
        json.Jsonize("block_size", blockSize, blockSize); // 未压缩块的大小
        json.Jsonize("compress_file_length", compressFileLen, compressFileLen); // 仅压缩数据的长度
        json.Jsonize("decompress_file_length", deCompressFileLen, deCompressFileLen); // 原始文件总长度
        json.Jsonize("max_compress_block_size", maxCompressBlockSize, maxCompressBlockSize);
        json.Jsonize("enable_compress_address_mapper", enableCompressAddressMapper, enableCompressAddressMapper);
        json.Jsonize("additional_information", additionalInfo, additionalInfo);
    }

public:
    std::string compressorName;
    size_t blockCount;
    size_t blockSize;
    size_t compressFileLen;
    size_t deCompressFileLen;
    size_t maxCompressBlockSize;
    bool enableCompressAddressMapper;
    util::KeyValueMap additionalInfo; // 用于存储扩展信息，如 hint, meta file 等
};
```

**设计剖析**:

*   **JSON 格式**: 采用 JSON 格式使得元数据易于阅读和调试，同时也具备良好的可扩展性。通过 `additionalInfo` 这个 `KeyValueMap`，可以轻松地向后兼容，添加新的配置项而无需修改核心结构。
*   **关键信息**: `blockSize` 和 `deCompressFileLen` 是进行逻辑偏移计算的基础。`blockCount` 和 `compressFileLen` 则用于定位和读取地址映射数据。`enableCompressAddressMapper` 标志位决定了地址映射数据是否被 `CompressFileAddressMapperEncoder` 压缩过。
*   **扩展性**: `additionalInfo` 是一个非常重要的设计。它被用来传递各种开关和参数，例如：
    *   `COMPRESS_ENABLE_META_FILE`: 是否使用独立的 `.meta` 文件存储地址映射信息。
    *   `COMPRESS_ENABLE_HINT_PARAM_NAME`: 是否启用了 Hint 优化。
    *   `address_mapper_data_size`: 地址映射数据的确切大小。
    *   `hint_data_size`: Hint 数据的大小。

### 2.2. `CompressFileAddressMapper`: 地址翻译核心

`CompressFileAddressMapper` 是实现随机访问的核心组件。它在内存中持有地址映射表，并提供了一系列方法来将逻辑偏移转换为物理块信息。

#### 2.2.1. 核心数据成员

*   `size_t* _baseAddr`: 指向地址映射数据在内存中的起始地址。这块内存可能来自 mmap 的文件，也可能是从磁盘读取后单独分配的。
*   `uint32_t _powerOf2`, `_powerMark`: 用于快速计算逻辑偏移所在的块索引。`blockSize` 必须是 2 的幂，`offset >> _powerOf2` 即可得到块索引，`offset & _powerMark` 得到块内偏移。这是性能优化的关键细节。
*   `size_t _blockCount`: 总块数。
*   `std::unique_ptr<CompressFileAddressMapperEncoder> _addrMapEncoder`: 地址映射编码/解码器。
*   `bool _isCompressed`: 标志地址映射数据是否被压缩。
*   `util::BitmapPtr _hintBitmap`: 如果地址映射被压缩且存在 Hint 数据，用位图来标记哪些块使用了 Hint。

#### 2.2.2. 读取初始化: `InitForRead`

`InitForRead` 的逻辑清晰地展示了如何根据 `CompressFileInfo` 重建地址映射。

```cpp
// file_system/file/CompressFileAddressMapper.cpp

void CompressFileAddressMapper::InitForRead(const std::shared_ptr<CompressFileInfo>& fileInfo,
                                            const std::shared_ptr<FileReader>& reader, IDirectory* directory)
{
    // 1. 从 fileInfo 初始化基本元数据
    InitMetaFromCompressInfo(fileInfo);

    // ... 省略 ResourceFile 相关逻辑 ...

    // 2. 加载地址映射数据
    InitAddressMapperData(fileInfo, reader, resource.get());
    // 3. 如果需要，加载 Hint 位图
    InitHintBitmap(fileInfo, reader, resource.get());
    // 4. 如果需要，加载 Hint 数据读取器
    InitHintDataReader(fileInfo, reader, resource.get());

    // ...
}
```

`InitAddressMapperData` 是关键步骤：

```cpp
// file_system/file/CompressFileAddressMapper.cpp
void CompressFileAddressMapper::InitAddressMapperData(const std::shared_ptr<CompressFileInfo>& fileInfo,
                                                      const std::shared_ptr<FileReader>& reader,
                                                      CompressMapperResource* resource) noexcept(false)
{
    // 判断是否使用独立的 .meta 文件
    bool useMetaFile = GetTypeValueFromKeyValueMap(fileInfo->additionalInfo, COMPRESS_ENABLE_META_FILE,
                                                   DEFAULT_COMPRESS_ENABLE_META_FILE);

    // 计算地址映射数据在文件中的起始偏移
    size_t beginOffset = useMetaFile ? 0 : fileInfo->compressFileLen;
    size_t mapperDataLength = GetAddressMapperDataSize();
    char* baseAddr = (char*)reader->GetBaseAddress();

    if (baseAddr != nullptr) {
        // 如果是 mmap 文件，直接指针赋值，零拷贝
        _baseAddr = (size_t*)(baseAddr + beginOffset);
    } else if (resource == nullptr) { // reuse resource file
        // ... 从已有的 ResourceFile 中获取
    } else {
        // 从磁盘读取到新分配的内存中
        resource->mapperBaseAddress = new char[mapperDataLength];
        resource->memoryUsed += mapperDataLength;
        if (mapperDataLength != reader->Read(resource->mapperBaseAddress, mapperDataLength, beginOffset).GetOrThrow()) {
            // ... 错误处理 ...
        }
        _baseAddr = (size_t*)resource->mapperBaseAddress;
    }
    // ...
}
```

**设计剖析**:

*   **mmap 优化**: `if (baseAddr != nullptr)` 的判断是核心性能优化点。对于 mmap 的文件，地址映射数据无需从磁盘 `read`，也无需额外分配内存，直接通过指针计算得到其在内存中的地址，实现了零拷贝和最低的延迟。
*   **布局灵活性**: 通过 `useMetaFile` 参数，支持两种物理布局。将元数据和地址映射放在独立文件（`.meta`）中，可以避免数据文件在追加写入时需要移动元数据块的尴尬，简化了文件管理。同时，对于只读或一次性写入的场景，将所有信息打包在单个文件中也同样支持。
*   **ResourceFile 机制**: `ResourceFile` 是 Indexlib 文件系统中的一个有趣设计，它允许多个文件读取器共享同一份内存资源（如这里加载的地址映射数据）。当多个 `CompressFileReader` 读取同一个压缩文件时，地址映射数据只需加载一次，后续的读取器可以直接复用，节省了内存和加载时间。

#### 2.2.3. 核心翻译方法

`CompressFileAddressMapper` 提供了一组高效的内联方法用于地址翻译。

```cpp
// file_system/file/CompressFileAddressMapper.h

size_t OffsetToBlockIdx(int64_t offset) const noexcept { return offset >> _powerOf2; }
size_t OffsetToInBlockOffset(int64_t offset) const noexcept { return offset & _powerMark; }

size_t CompressBlockAddress(int64_t blockIdx) const noexcept
{
    assert((size_t)blockIdx <= _blockCount);
    if (_isCompressed) {
        // 如果被压缩，调用解码器
        assert(_addrMapEncoder);
        return _addrMapEncoder->GetOffset((char*)_baseAddr, blockIdx);
    }
    // 否则，直接数组访问
    return _baseAddr[blockIdx] & OFFSET_MASK;
}

size_t CompressBlockLength(int64_t blockIdx) const noexcept
{
    assert((size_t)blockIdx < _blockCount);
    // 块的长度等于下一个块的起始地址减去当前块的起始地址
    return CompressBlockAddress(blockIdx + 1) - CompressBlockAddress(blockIdx);
}
```

**设计剖析**:

*   **位运算**: `OffsetToBlockIdx` 和 `OffsetToInBlockOffset` 使用位运算，这是在 `blockSize` 为 2 的幂次方的约束下能达到的最高效率。
*   **透明解码**: `CompressBlockAddress` 方法内部处理了地址映射是否被压缩的情况。如果 `_isCompressed` 为 true，它会调用 `_addrMapEncoder->GetOffset` 进行实时解码；否则，直接访问数组。这种判断对上层调用者是透明的。
*   **巧妙的长度计算**: `CompressBlockLength` 的计算方式非常巧妙。地址映射表实际上存储的是每个块的**累积结束位置**。因此，第 `i` 个块的长度就是 `_baseAddr[i+1] - _baseAddr[i]`。这避免了为每个块额外存储一个长度字段，节省了空间。

### 2.3. `CompressFileAddressMapperEncoder`: 空间压缩的艺术

当文件块数非常多时，存储每个块的 64 位（`size_t`）结束偏移会占用相当大的空间。例如，一个 1GB 的文件，如果块大小为 4KB，将有 262144 个块，地址映射表本身就需要 `262145 * 8` 字节，约 2MB。`CompressFileAddressMapperEncoder` 的目标就是减少这部分开销。

它采用了一种基于**增量编码**的简单压缩算法。

```cpp
// file_system/file/CompressFileAddressMapperEncoder.cpp

bool CompressFileAddressMapperEncoder::Encode(size_t* offsetVec, size_t num, char* outputBuffer,
                                              size_t outputBufferSize) noexcept
{
    // ...
    size_t base = 0;
    char* cursor = outputBuffer;
    for (size_t i = 0; i < num; i++) {
        if (i % _maxBatchNum == 0) {
            // 每 _maxBatchNum 个块，存储一个完整的 64 位基准偏移
            base = offsetVec[i];
            *(size_t*)cursor = base;
            cursor += sizeof(size_t);
        } else {
            // 批次内的其他块，存储与基准偏移的差值（delta）
            if (offsetVec[i] < base) {
                return false;
            }
            size_t delta = offsetVec[i] - base;
            if (delta > numeric_limits<uint16_t>::max()) {
                // 如果差值太大，无法用 16 位表示，则编码失败
                return false;
            }
            // 将差值用 16 位整数存储
            *(uint16_t*)cursor = (uint16_t)delta;
            cursor += sizeof(uint16_t);
        }
    }
    return true;
}
```

**设计剖析**:

*   **批处理增量编码**: 算法将所有块分为大小为 `_maxBatchNum`（默认为 16）的批次。每个批次的第一个块存储其完整的 64 位偏移作为 `base`。批次内后续的 15 个块，只存储它们相对于 `base` 的 16 位增量（`delta`）。
*   **空间效率**: 这种方法将原本需要 `16 * 8 = 128` 字节的空间，压缩到了 `8 + 15 * 2 = 38` 字节，压缩率约为 70%。
*   **解码效率**: 解码同样高效。`GetOffset` 方法首先计算目标 `idx` 所在的批次和批内索引，读取 `base` 值，然后加上对应的 `delta` 值即可。由于没有复杂的解压算法，解码速度极快。
*   **约束与权衡**: 该算法有一个前提假设：一个批次内（16个块）的累积压缩数据大小不会超过 `uint16_t` 的最大值（65535）。对于大多数压缩算法和数据类型，这个假设是成立的。如果数据几乎不可压缩，导致 `delta` 超出范围，编码会失败，系统将回退到不压缩地址映射的模式。这是一个非常务实的设计权衡。

## 3. 技术风险与权衡

1.  **元数据一致性**: 系统依赖于三个部分来保证正确读取：数据文件、`.info` 文件、可能的 `.meta` 文件。任何一个部分损坏或不一致都会导致读取失败。例如，如果数据文件被修改，而 `.info` 文件未更新，`compressFileLen` 就会出错，导致地址映射加载失败。系统没有内置的校验和（checksum）或版本强校验机制，依赖于上层文件发布和管理系统来保证原子性和一致性。

2.  **`blockSize` 的选择**: 如前所述，`blockSize` 是一个至关重要的调优参数。它直接影响压缩率、随机访问性能和内存开销。
    *   小 `blockSize` (e.g., 4KB, 8KB): 随机访问延迟低，内存占用少。但压缩率可能较低（因为压缩字典小），元数据开销大。
    *   大 `blockSize` (e.g., 256KB, 1MB): 压缩率高，元数据开销小。但随机访问时，即使只读 1KB 数据也需要解压整个大块，造成 CPU 和内存浪费，延迟较高。
    *   **决策**: 需要根据应用的访问模式（顺序 vs. 随机）、数据特征（重复度）和硬件资源（CPU, 内存）进行综合测试和选择。

3.  **地址映射编码的局限性**: `CompressFileAddressMapperEncoder` 的增量编码虽然高效，但对于压缩率极低的数据可能会失效。在这种罕见情况下，系统会回退到不压缩模式，牺牲一些空间来保证可用性。这是一种优雅降级（Graceful Degradation）的策略。

## 4. 结论

Indexlib 的元数据与地址映射系统是其高性能压缩框架的基石。通过 `CompressFileInfo` 提供的顶层元数据、`CompressFileAddressMapper` 实现的高效地址翻译，以及 `CompressFileAddressMapperEncoder` 带来的巧妙空间优化，该系统成功地在压缩文件的空间效率和随机访问性能之间取得了出色的平衡。对 mmap 模式的特殊优化、对不同物理布局的支持以及 `ResourceFile` 的共享机制，进一步展示了其设计的成熟度和对不同应用场景的深刻理解。掌握这套机制的原理，是理解和优化 Indexlib 存储性能的关键所在。
