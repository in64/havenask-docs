# WAL 类定义与接口设计

**涉及文件**:
- `file_system/wal/Wal.h`

## 概述

`Wal.h` 文件是 Havenask 存储引擎中 Write-Ahead Log (WAL) 模块的头文件，它定义了 WAL 系统的核心类 `WAL` 及其辅助类 `WALOperator`、`Writer` 和 `Reader`。这个文件不仅声明了 WAL 模块的公共接口，还通过其内部结构和成员函数的设计，揭示了 WAL 系统如何实现数据持久化、崩溃恢复以及高效读写的架构蓝图。它是理解 WAL 模块整体功能和协作方式的关键入口。

## 设计目标与技术选型

### 核心目标

`Wal.h` 中定义的类和接口旨在实现以下核心目标：

1.  **模块化与职责分离**: 将 WAL 的不同功能（如整体管理、写入、读取、压缩处理）封装到独立的类中，提高代码的可维护性和可测试性。
2.  **数据持久化与恢复**: 提供一套完整的接口，支持将数据记录追加到 WAL 文件中，并在系统重启后能够从 WAL 中恢复数据，确保数据一致性。
3.  **高性能读写**: 通过内部缓冲区、直接 I/O（可选）和压缩机制，优化 WAL 文件的读写性能，以满足高吞吐量场景的需求。
4.  **容错性与健壮性**: 考虑文件操作失败、数据损坏等异常情况，提供错误处理和重试机制，增强系统的稳定性。
5.  **可配置性**: 允许通过 `WALOption` 配置 WAL 的行为，如工作目录、压缩类型、I/O 模式等，以适应不同的部署环境和性能要求。

### 技术栈选择与设计动机

*   **C++**: 作为 Havenask 的主要开发语言，C++ 提供了高性能和底层控制能力，非常适合实现对文件系统进行精细操作的 WAL 模块。
*   **面向对象设计**: 采用类和对象的封装、继承、多态等特性，将复杂的 WAL 逻辑分解为更小、更易于管理的单元。例如，`Writer` 和 `Reader` 类分别专注于写入和读取逻辑，而 `WALOperator` 则处理通用的操作，如压缩器管理。
*   **`std::shared_ptr` 和 `std::unique_ptr`**: 广泛使用智能指针管理内存，避免手动内存管理带来的内存泄漏和悬垂指针问题，提高代码的健壮性。
*   **`fslib` 库**: 依赖 `fslib` 库进行底层文件系统操作。`fslib` 是 Havenask 内部的文件系统抽象层，能够屏蔽不同存储介质（如本地文件系统、HDFS、OSS 等）的差异，提供统一的文件操作接口。这使得 WAL 模块能够轻松地部署在不同的存储环境中。
*   **`indexlib::util::buffer_compressor`**: 引入 Havenask 内部的缓冲区压缩库，支持多种压缩算法（ZLIB, SNAPPY, LZ4, ZSTD 等）。这使得 WAL 能够根据配置对记录数据进行压缩，有效减少存储空间和 I/O 带宽。
*   **Protobuf 定义的元数据**: 结合 `Wal.proto` 中定义的 `WalRecordMeta`，WAL 模块能够以结构化的方式存储每条记录的元数据（如偏移量、CRC、数据长度、压缩类型），这对于高效解析和验证 WAL 记录至关重要。

## 核心类结构解析

`Wal.h` 中定义了以下几个核心类：

### 1. `WAL` 类

`WAL` 类是整个 WAL 模块的入口和核心管理器，负责 WAL 文件的生命周期管理、记录的追加和读取，以及崩溃恢复的协调。

```cpp
class WAL
{
public:
    enum class CompressType { /* ... */ };
    struct WALOption { /* ... */ };

    static bool IsValidWALDirName(const std::string& fileName);

    explicit WAL(const WALOption& opt);
    ~WAL() = default;

    WAL(const WAL&) = delete;
    WAL& operator=(const WAL&) = delete;

    bool Init();
    bool AppendRecord(const std::string& record);
    bool ReadRecord(std::string& record);
    bool IsRecovered() const { return _isRecovered; };

    bool ReOpen(bool removeOldFile = false);
    bool SeekRecoverPosition(int64_t recoverPos);
    bool Recycle();

public:
    // Nested classes: WALOperator, Writer, Reader

private:
    // Constants and member variables
    static constexpr char wal_prefix[] = "log_";
    static constexpr char wal_suffix[] = ".__wal__";
    static constexpr uint32_t MAX_WAL_FILE_SIZE = 1 * 1024 * 1024 * 1024; // 1G
    static constexpr int64_t DEFAULT_RECOVER_POSITION = -1;
    static constexpr size_t CONTINUOUS_APPEND_FAIL_LIMITS = 50;

    std::pair<std::shared_ptr<fslib::fs::File>, size_t> OpenFile(size_t offset, bool useDirectIO, fslib::Flag mode);
    bool OpenWriter(size_t offset);
    bool OpenReader(size_t begin_pos, size_t offset);
    bool LoadWALFiles();

private:
    const WALOption _walOpt;
    std::map<uint64_t, fslib::RichFileMeta> _walFiles; // map<offset, meta>
    bool _isRecovered = false;
    std::unique_ptr<Writer> _writer;
    std::unique_ptr<Reader> _reader;
    size_t _continuousAppendFails;

    AUTIL_LOG_DECLARE();
};
```

*   **功能**: 管理 WAL 文件的创建、打开、关闭、记录追加、记录读取以及恢复逻辑。它协调 `Writer` 和 `Reader` 对象来执行实际的 I/O 操作。
*   **核心成员**: 
    *   `_walOpt`: `WALOption` 结构体，存储 WAL 的配置选项，如工作目录、压缩类型、I/O 模式等。
    *   `_walFiles`: `std::map<uint64_t, fslib::RichFileMeta>`，维护当前 WAL 目录中所有 WAL 文件的元数据，键是文件的起始偏移量，值是文件的 `fslib::RichFileMeta`（包含文件大小等信息）。这对于管理多个 WAL 文件和恢复过程中的文件切换至关重要。
    *   `_writer`: `std::unique_ptr<Writer>`，当前活跃的 WAL 写入器。当 WAL 文件达到最大大小时，会切换到新的 WAL 文件，并创建新的 `Writer`。
    *   `_reader`: `std::unique_ptr<Reader>`，当前活跃的 WAL 读取器，用于从 WAL 文件中读取记录进行恢复。
    *   `_isRecovered`: 标志，指示 WAL 是否已经完成恢复。
    *   `_continuousAppendFails`: 连续追加失败计数器，用于触发 WAL 文件的重新打开或错误处理。
*   **关键方法**: 
    *   `Init()`: 初始化 WAL 模块，包括创建工作目录、加载现有 WAL 文件、确定恢复位置并打开写入器。
    *   `AppendRecord(const std::string& record)`: 将一条记录追加到 WAL 中。如果当前 WAL 文件达到最大大小，会自动切换到新的 WAL 文件。
    *   `ReadRecord(std::string& record)`: 从 WAL 中读取一条记录，用于恢复。如果当前文件读取完毕，会自动切换到下一个 WAL 文件。
    *   `SeekRecoverPosition(int64_t recoverPos)`: 根据指定的恢复位置打开合适的 WAL 文件和读取器。
    *   `ReOpen(bool removeOldFile = false)`: 关闭当前写入器并打开一个新的写入器，可选地删除旧文件。用于 WAL 文件轮转或错误恢复。
    *   `Recycle()`: 清理 WAL 工作目录中的所有 WAL 文件，通常在 WAL 数据不再需要时调用。
    *   `OpenFile()`, `OpenWriter()`, `OpenReader()`, `LoadWALFiles()`: 内部辅助方法，用于文件操作和 WAL 文件管理。
*   **设计动机**: `WAL` 类作为顶层协调者，封装了 WAL 系统的复杂性，对外提供简洁的接口。通过管理多个 WAL 文件，实现了日志的无限增长和分段存储。错误计数器和自动文件切换机制增强了系统的健壮性。

### 2. `WAL::WALOperator` 类

`WALOperator` 是一个静态辅助类，主要负责管理和提供压缩器实例，以及 WAL 内部 `CompressType` 和 Protobuf 定义的 `CompressType` 之间的转换。

```cpp
class WAL::WALOperator
{
public:
    WALOperator() = default;
    virtual ~WALOperator() = default;

    static bool Init();
    static const indexlib::util::BufferCompressorPtr GetCompressor(const WAL::CompressType& type);

    static bool ToPBType(const WAL::CompressType& src, ::indexlib::proto::CompressType& dst);
    static bool ToWALType(const ::indexlib::proto::CompressType& src, WAL::CompressType& type);

private:
    static std::map<WAL::CompressType, indexlib::util::BufferCompressorPtr> _compressors;

    AUTIL_LOG_DECLARE();
};
```

*   **功能**: 提供 WAL 模块通用的操作，特别是与数据压缩相关的服务。
*   **核心成员**: 
    *   `static std::map<WAL::CompressType, indexlib::util::BufferCompressorPtr> _compressors`: 静态成员，存储不同压缩类型的 `BufferCompressor` 实例。这些压缩器在系统启动时初始化，并在整个 WAL 模块生命周期中复用，避免重复创建开销。
*   **关键方法**: 
    *   `static bool Init()`: 初始化所有支持的压缩器实例。
    *   `static const indexlib::util::BufferCompressorPtr GetCompressor(const WAL::CompressType& type)`: 根据压缩类型获取对应的压缩器实例。
    *   `static bool ToPBType(...)`, `static bool ToWALType(...)`: 负责 WAL 内部枚举类型和 Protobuf 定义枚举类型之间的转换，确保数据在序列化和反序列化时类型匹配。
*   **设计动机**: 将压缩器管理和类型转换逻辑集中到一个静态类中，实现了代码复用和职责分离。静态成员确保压缩器只被初始化一次，提高了效率。

### 3. `WAL::Writer` 类

`Writer` 类负责将记录写入 WAL 文件，包括数据压缩、元数据封装和实际的文件 I/O 操作。

```cpp
class WAL::Writer : public WALOperator
{
public:
    explicit Writer(const std::shared_ptr<fslib::fs::File> file, uint64_t beginOffset,
                        const WAL::CompressType& type);
    ~Writer();

    Writer(const Writer&) = delete;
    Writer& operator=(const Writer&) = delete;

    bool AppendRecord(const std::string& record);

    size_t LastOffset() const { return _offset; }
    size_t LastNextFileOffset() const { return _offset + _beginOffset; }
    const char* GetWriterFileName() const { return _file->getFileName(); }

private:
    uint32_t GetDataCRC(const std::string& record);
    bool Compress(const std::string& record, std::string& compressData);

private:
    static constexpr uint32_t DEFAULT_DATA_BUF_SIZE = 1024 * 1024; // 1M
    std::unique_ptr<std::vector<char>> _buffer;
    std::shared_ptr<fslib::fs::File> _file;
    const uint64_t _beginOffset;
    const WAL::CompressType _compressType;
    size_t _offset = 0;

    AUTIL_LOG_DECLARE();
};
```

*   **功能**: 专注于 WAL 记录的写入操作。它接收原始记录数据，进行压缩、计算 CRC、封装 Protobuf 元数据，然后写入到底层文件。
*   **核心成员**: 
    *   `std::shared_ptr<fslib::fs::File> _file`: 当前写入的 WAL 文件句柄。
    *   `std::unique_ptr<std::vector<char>> _buffer`: 内部写入缓冲区，用于暂存待写入的数据，减少系统调用次数，提高写入效率。
    *   `_beginOffset`: 当前 WAL 文件在整个逻辑 WAL 中的起始偏移量。
    *   `_compressType`: 当前写入器使用的压缩类型。
    *   `_offset`: 当前写入器在当前 WAL 文件中的偏移量。
*   **关键方法**: 
    *   `AppendRecord(const std::string& record)`: 核心写入方法。它将记录数据进行压缩，计算 CRC，构建 `WalRecordMeta`，然后将元数据和压缩数据写入 `_file`。
    *   `GetDataCRC(const std::string& record)`: 计算记录数据的 CRC32C 校验和。
    *   `Compress(const std::string& record, std::string& compressData)`: 对记录数据进行压缩。
    *   `LastOffset()`, `LastNextFileOffset()`: 提供当前写入偏移量和下一个 WAL 文件的起始偏移量，供 `WAL` 类进行文件轮转决策。
*   **设计动机**: `Writer` 类将复杂的写入逻辑（压缩、CRC、Protobuf 序列化、文件 I/O）封装起来，对外提供简单的 `AppendRecord` 接口。内部缓冲区的使用是典型的性能优化手段。

### 4. `WAL::Reader` 类

`Reader` 类负责从 WAL 文件中读取记录，包括数据解压缩、元数据解析和文件 I/O 操作。

```cpp
class WAL::Reader : public WALOperator
{
public:
    explicit Reader(const std::shared_ptr<fslib::fs::File> file, size_t fileLength, bool checksum,
                        size_t skipOffset, uint32_t beginOffset);
    ~Reader() { _file->close(); }

    Reader(const Reader&) = delete;
    Reader& operator=(const Reader&) = delete;

    bool ReadRecord(std::string& record);
    size_t BeginOffset() const { return _beginOffset; }
    size_t LastRecordOffset() const { return _offsetInFile + _beginOffset; }
    bool IsEof() const { return _eof; }

private:
    bool DeCompress(const WAL::CompressType& type, const char* rawData, uint32_t len, std::string& record) const;
    bool RotateNextBlock(size_t& leftLen, int64_t requiredBytes);

private:
    static constexpr uint32_t DEFAULT_DATA_BUF_SIZE = 1024 * 1024; // 1M
    std::shared_ptr<fslib::fs::File> _file;
    size_t _fileLength = 0;
    std::unique_ptr<std::vector<char>> _buffer;
    bool _eof = false;
    size_t _endOffsetInBlock = 0;
    size_t _offsetInBlock = 0;
    size_t _offsetInFile = 0;
    size_t _skipOffset;
    const uint32_t _headerLen;
    const uint64_t _beginOffset;
    const bool _isCheckSum;

    AUTIL_LOG_DECLARE();
};
```

*   **功能**: 专注于 WAL 记录的读取操作。它从底层文件读取数据，解析 Protobuf 元数据，进行 CRC 校验，然后解压缩数据。
*   **核心成员**: 
    *   `std::shared_ptr<fslib::fs::File> _file`: 当前读取的 WAL 文件句柄。
    *   `_fileLength`: 当前 WAL 文件的总长度。
    *   `std::unique_ptr<std::vector<char>> _buffer`: 内部读取缓冲区，用于从文件中预读数据，减少系统调用次数。
    *   `_eof`: 标志，指示是否已到达文件末尾。
    *   `_offsetInBlock`, `_endOffsetInBlock`: 缓冲区内部的读写偏移量。
    *   `_offsetInFile`: 当前读取器在当前 WAL 文件中的逻辑偏移量。
    *   `_skipOffset`: 在当前文件中跳过的字节数，用于从指定位置开始恢复。
    *   `_isCheckSum`: 是否进行 CRC 校验。
*   **关键方法**: 
    *   `ReadRecord(std::string& record)`: 核心读取方法。它从 `_file` 中读取数据，解析 `WalRecordMeta`，进行 CRC 校验，然后解压缩数据到 `record`。
    *   `DeCompress(...)`: 对读取到的数据进行解压缩。
    *   `RotateNextBlock(...)`: 当缓冲区数据不足时，从文件中读取更多数据填充缓冲区，并处理缓冲区内部的偏移量。
    *   `IsEof()`: 判断是否已读到文件末尾。
*   **设计动机**: `Reader` 类将复杂的读取逻辑（文件 I/O、Protobuf 反序列化、CRC 校验、解压缩）封装起来。内部缓冲区和 `RotateNextBlock` 机制是典型的预读优化，旨在提高读取效率。

## 系统架构与关键实现细节

`Wal.h` 中定义的类共同构建了一个分层、模块化的 WAL 系统架构：

1.  **顶层管理 (`WAL`)**: `WAL` 类作为整个模块的协调者，负责 WAL 文件的整体生命周期管理、文件轮转、恢复流程的控制。它不直接进行文件 I/O，而是委托给 `Writer` 和 `Reader`。
2.  **I/O 封装 (`Writer`, `Reader`)**: `Writer` 和 `Reader` 类分别封装了 WAL 记录的写入和读取细节，包括数据压缩/解压缩、Protobuf 元数据处理、CRC 校验以及与 `fslib` 的交互。它们内部使用缓冲区来优化 I/O 性能。
3.  **通用操作 (`WALOperator`)**: `WALOperator` 提供通用的辅助功能，如压缩器管理和类型转换，这些功能被 `Writer` 和 `Reader` 复用。
4.  **配置 (`WALOption`)**: `WALOption` 结构体提供了灵活的配置机制，允许用户根据需求调整 WAL 的行为，例如选择不同的压缩算法、启用/禁用直接 I/O 和校验和。

**关键实现细节**:

*   **文件命名约定**: WAL 文件名遵循 `log_` + `offset` + `.__wal__` 的格式（例如 `log_0.__wal__`, `log_1073741824.__wal__`），其中 `offset` 是该文件在整个逻辑 WAL 中的起始字节偏移量。这种命名方式使得 `WAL` 类能够方便地管理和查找文件，尤其是在恢复时。
*   **文件轮转**: 当当前写入的 WAL 文件大小达到 `MAX_WAL_FILE_SIZE` (1GB) 时，`WAL::AppendRecord` 会触发 `WAL::OpenWriter` 创建一个新的 WAL 文件，并从新的文件开始写入。这避免了单个 WAL 文件过大，便于管理和清理。
*   **恢复机制**: `WAL::SeekRecoverPosition` 方法根据指定的恢复点（逻辑偏移量）找到对应的 WAL 文件和文件内偏移量，然后创建 `Reader` 从该位置开始读取。`WAL::ReadRecord` 方法在读取到当前文件末尾时，会自动切换到下一个 WAL 文件，实现连续恢复。
*   **数据完整性**: `Writer` 在写入前计算数据的 CRC32C 校验和并存储在 `WalRecordMeta` 中，`Reader` 在读取后进行校验。这确保了数据在存储和传输过程中的完整性。
*   **缓冲区优化**: `Writer` 和 `Reader` 都使用内部缓冲区 (`_buffer`) 来批量读写数据，减少了频繁的 `fslib` 调用，从而提高了 I/O 吞吐量。

## 潜在的技术风险与考量

1.  **文件句柄管理**: `Writer` 和 `Reader` 持有 `fslib::fs::File` 的 `shared_ptr`。虽然智能指针管理了生命周期，但在高并发或异常退出时，需要确保文件句柄的及时关闭和资源释放，防止文件泄漏或文件系统资源耗尽。
2.  **I/O 性能瓶颈**: 尽管有缓冲区和直接 I/O 选项，但 WAL 的性能仍然受限于底层存储介质的 I/O 能力。在极端写入负载下，磁盘 I/O 可能会成为瓶颈。需要持续监控和优化。
3.  **错误处理与重试**: `AppendRecord` 中的 `_continuousAppendFails` 计数器和 `ReOpen` 机制提供了一定的容错能力，但对于更复杂的错误场景（如磁盘满、文件系统损坏），需要更完善的错误处理策略和告警机制。
4.  **恢复效率**: `SeekRecoverPosition` 依赖于文件命名中的偏移量来定位文件。如果 WAL 文件数量巨大，`LoadWALFiles` 遍历目录和 `std::map` 的查找可能会有性能开销。对于超大规模的 WAL，可能需要更高效的文件索引机制。
5.  **内存使用**: `Writer` 和 `Reader` 的内部缓冲区默认大小为 1MB。在高并发场景下，如果存在大量 `Writer` 或 `Reader` 实例，可能会占用较多内存。需要根据实际内存限制和性能需求进行调整。
6.  **并发安全**: `Wal.h` 中没有明确的锁机制。如果 `WAL` 实例的 `AppendRecord` 或 `ReadRecord` 方法可能被多个线程同时调用，需要外部同步机制来保证线程安全。

## 总结

`Wal.h` 文件通过清晰的类定义和接口设计，为 Havenask WAL 模块构建了一个健壮、高效且可扩展的框架。它将复杂的 WAL 逻辑分解为可管理的组件，并利用智能指针、缓冲区优化和 `fslib` 抽象层等技术，实现了可靠的数据持久化和恢复功能。深入理解 `Wal.h` 中定义的类及其协作方式，是掌握 Havenask WAL 模块内部工作原理的关键一步。
