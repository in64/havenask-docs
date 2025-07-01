
# Indexlib 文件系统：装饰器与功能扩展目录分析

**涉及文件:**
*   `file_system/FenceDirectory.h`
*   `file_system/FenceDirectory.cpp`
*   `file_system/LinkDirectory.h`
*   `file_system/LinkDirectory.cpp`

## 1. 引言

在 Indexlib 强大而灵活的文件系统之上，还构建了一系列通过装饰器模式来扩展核心功能的目录实现。这些实现不直接管理存储，而是包装（“装饰”）一个已有的 `IDirectory` 对象，为其赋予新的行为和能力。本文将深入剖析两种重要的功能扩展目录：`FenceDirectory` 和 `LinkDirectory`。`FenceDirectory` 为分布式环境下的写入操作提供了关键的隔离和原子性保证，而 `LinkDirectory` 则提供了一种高效的、只读的目录访问机制。理解它们的设计和实现，对于掌握 Indexlib 在复杂场景下的数据管理策略至关重要。

## 2. `FenceDirectory`: 分布式写入的安全卫士

在分布式系统中，多个写入者（Worker）可能同时尝试更新同一份数据（例如，一个索引分片）。如果缺乏有效的并发控制，就可能导致数据损坏或状态不一致。`FenceDirectory` 的核心使命就是解决这个问题，它通过引入 “Fence” 机制（也常被称为 Epoch Fencing 或 Generation Fencing），确保在任何时刻，只有一个合法的写入者能够成功提交数据。

### 2.1. 设计理念：临时写入 + 原子提交

`FenceDirectory` 的工作原理可以概括为 “先写入临时目录，再原子性地重命名提交”。它通过装饰一个底层的 `DirectoryImpl` 对象，拦截了所有的写操作，并将其重定向到一个临时的、与当前写入者 epoch 相关的路径下。只有当所有写入操作成功完成并关闭后，这些临时文件才会被原子性地 `rename` 到最终的目标位置。

这个过程的核心要素是 `FenceContext`，它包含了当前写入者的 `epochId`。`epochId` 是一个单调递增的ID，由外部的 Master（如 suez）统一分配。只有持有最新 `epochId` 的写入者才能成功地执行 `rename` 和 `remove` 等关键操作。

整个流程如下：

1.  **初始化:** 创建 `FenceDirectory` 时，需要提供一个被包装的 `IDirectory` 和一个 `FenceContext`。
2.  **准备临时目录:** 当第一次发起写操作（如 `CreateFileWriter` 或 `MakeDirectory`）时，`FenceDirectory` 会在被包装的目录下创建一个名为 `pre.__tmp__/` 的特殊临时目录（如果尚不存在）。
3.  **重定向写入:** 所有的文件和目录创建请求都会被重定向到这个临时目录下，并且文件名或目录名会被附加一个后缀，即当前的 `epochId`。例如，`CreateFileWriter("version.0")` 会实际在 `pre.__tmp__/version.0.epoch_123` 创建文件。
4.  **返回临时句柄:** `FenceDirectory` 会返回一个 `TempFileWriter` 或 `TempDirectory` 对象给上层。这些临时对象持有一个回调函数。
5.  **提交:** 当上层调用 `TempFileWriter::Close()` 或 `TempDirectory::Close()` 时，会触发这个回调函数。该回调函数的核心逻辑是执行一个带 `FenceContext` 的 `rename` 操作，将 `pre.__tmp__/version.0.epoch_123` 原子性地移动到最终目标 `version.0`。底层的 `IFileSystem`（如 Pangu）会利用 `FenceContext` 中的 `epochId` 来保证 `rename` 的原子性和隔离性。如果此时有另一个持有旧 `epochId` 的 Worker 尝试操作，其 `rename` 会失败。

### 2.2. 关键代码解读 (`FenceDirectory.cpp`)

```cpp
// 构造函数：保存被包装的目录和 FenceContext
FenceDirectory::FenceDirectory(const std::shared_ptr<IDirectory>& rootDirectory,
                               const std::shared_ptr<FenceContext>& fenceContext, FileSystemMetricsReporter* reporter)
    : _directory(std::dynamic_pointer_cast<DirectoryImpl>(rootDirectory))
    , _fenceContext(fenceContext)
    , _reporter(reporter)
{
    assert(_fenceContext);
    assert(_directory);
}

// 创建文件写入器：核心的写操作拦截逻辑
FSResult<std::shared_ptr<FileWriter>> FenceDirectory::CreateFileWriter(const std::string& fileName,
                                                                       const WriterOption& writerOption) noexcept
{
    // ... 省略 slice writer 的处理 ...

    // 1. 确保临时目录已准备好
    if (_preDirectory == nullptr) {
        RETURN2_IF_FS_ERROR(PreparePreDirectory().Code(), std::shared_ptr<FileWriter>(), "PreparePreDirectory failed");
    }

    // 2. 在临时目录下创建带 epoch 后缀的文件
    auto [ec, file] = _preDirectory->CreateFileWriter(fileName + "." + _fenceContext->epochId, writerOption);
    RETURN2_IF_FS_ERROR(ec, file, "CreateFileWriter failed");

    // 3. 创建回调函数，用于关闭时提交 (rename)
    auto closeFunc = [this](const std::string& fileName) mutable noexcept { return this->CloseTempPath(fileName); };

    // 4. 返回 TempFileWriter，包装了真正的 writer 和回调
    return {FSEC_OK, std::make_shared<TempFileWriter>(file, move(closeFunc))};
}

// 创建目录：逻辑与 CreateFileWriter 类似
FSResult<std::shared_ptr<IDirectory>> FenceDirectory::MakeDirectory(const std::string& dirPath,
                                                                    const DirectoryOption& directoryOption) noexcept
{
    // ... 省略对已存在目录的处理 ...

    if (_preDirectory == nullptr) {
        RETURN2_IF_FS_ERROR(PreparePreDirectory().Code(), std::shared_ptr<IDirectory>(), "PreparePreDirectory failed");
    }

    string dir = dirPath + "." + _fenceContext->epochId;
    auto [ec2, directory2] = _preDirectory->MakeDirectory(dir, directoryOption);
    RETURN2_IF_FS_ERROR(ec2, directory2, "MakeDirectory [%s] failed", dir.c_str());

    auto closeFunc = [this](const std::string& dirName) mutable noexcept -> FSResult<void> {
        return this->CloseTempPath(dirName);
    };
    // 返回 TempDirectory
    return {FSEC_OK, std::make_shared<TempDirectory>(directory2, std::move(closeFunc), dir, _reporter)};
}

// 提交操作：执行带有 FenceContext 的 rename
FSResult<void> FenceDirectory::CloseTempPath(const std::string& name) noexcept
{
    // root/pre/xxx.epochId -> root/xxx
    string destName = name;
    autil::StringUtil::replaceLast(destName, "." + _fenceContext->epochId, "");
    return _preDirectory->InnerRename(name, _directory, destName, _fenceContext.get());
}

// 删除操作：将 FenceContext 传递给底层目录
FSResult<void> FenceDirectory::InnerRemoveFile(const std::string& filePath, const RemoveOption& removeOption) noexcept
{
    RemoveOption newRemoveOption(removeOption);
    newRemoveOption.fenceContext = _fenceContext.get();
    return _directory->InnerRemoveFile(filePath, newRemoveOption);
}
```

代码清晰地展示了 `FenceDirectory` 如何通过拦截写请求、附加 epoch、返回临时句柄和最终 `rename` 来实现其隔离机制。`CloseTempPath` 是整个机制的核心，它将临时文件“转正”，完成了事务的提交。

### 2.3. 技术风险与考量

*   **依赖底层原子性:** `FenceDirectory` 的正确性严重依赖于底层文件系统 `rename` 操作的原子性和 Fencing 支持。如果底层系统不支持，`FenceDirectory` 将无法提供有效的隔离保证。
*   **临时文件残留:** 如果在 `rename` 提交之前进程异常退出，`pre.__tmp__/` 目录下可能会残留临时文件。需要有配套的垃圾回收（GC）机制来定期清理这些无用的临时文件。
*   **性能:** 每次写操作都涉及到路径拼接和最终的 `rename`，可能会引入微小的性能开销。但在分布式环境下，数据一致性和正确性通常是首要考虑的因素。

## 3. `LinkDirectory`: 高效的只读视图

在 Indexlib 的一些场景中，特别是 Realtime (RT) 索引，需要频繁地访问已经生成并刷盘（flushed）的索引段。这些段的数据是不可变的。为了高效地访问这些数据，同时避免在内存中保留多余的副本，Indexlib 引入了 `LinkDirectory`。

### 3.1. 设计理念：符号链接的抽象

`LinkDirectory` 的核心思想类似于文件系统中的符号链接（Symbolic Link）。Indexlib 在启动时，可以配置一个全局的 `root_link` 路径。当一个段被刷盘后，`IFileSystem` 会在这个 `root_link` 目录下创建一个与原始段路径结构相同的“链接”目录结构。`LinkDirectory` 就是用来访问这些链接目录的。

它的主要特点是 **只读**。`LinkDirectory` 禁止了所有写操作，如 `MakeDirectory` 和 `CreateFileWriter`（`SLICE` 类型的写入除外，这通常用于内部的、非持久化的数据交换）。

### 3.2. 实现与应用

`LinkDirectory` 继承自 `LocalDirectory`，因为它本质上也是对 `IFileSystem` 的一层封装。它的特殊之处在于：

1.  **路径:** 它的根路径 `_rootDir` 指向的是 `root_link` 下的某个路径。
2.  **只读限制:** 它重写了 `MakeDirectory` 和 `DoCreateFileWriter` 方法，直接返回 `FSEC_NOTSUP`（不支持）错误，从而实现了只读的语义。

`LinkDirectory` 的主要应用场景是在 RT 索引中，当一个内存中的 RT Segment 被刷盘后，查询引擎可以通过 `LinkDirectory` 来访问这个新生成的磁盘上的 Segment，而无需重新加载数据或复制文件。这提供了一种从内存视图到磁盘视图的平滑、高效的切换方式。

### 3.3. 关键代码解读 (`LinkDirectory.cpp`)

```cpp
// 构造函数：路径必须包含 root_link 标识，并确保路径存在
LinkDirectory::LinkDirectory(const std::string& path, const std::shared_ptr<IFileSystem>& fileSystem)
    : LocalDirectory(path, fileSystem)
{
    assert(path.find(FILE_SYSTEM_ROOT_LINK_NAME) != std::string::npos);
    assert(GetFileSystem()->IsExist(path).GetOrThrow());
}

// 禁止创建目录
FSResult<std::shared_ptr<IDirectory>> LinkDirectory::MakeDirectory(const std::string& dirPath,
                                                                   const DirectoryOption& directoryOption) noexcept
{
    AUTIL_LOG(ERROR, "LinkDirectory [%s] not support MakeDirectory", GetRootDir().c_str());
    return {FSEC_NOTSUP, std::shared_ptr<IDirectory>()};
}

// 禁止创建文件写入器（SLICE 类型除外）
FSResult<std::shared_ptr<FileWriter>> LinkDirectory::DoCreateFileWriter(const std::string& filePath,
                                                                        const WriterOption& writerOption) noexcept
{
    if (writerOption.openType == FSOT_SLICE) {
        return LocalDirectory::DoCreateFileWriter(filePath, writerOption);
    }
    AUTIL_LOG(ERROR, "LinkDirectory [%s] not support CreateFileWriter", GetRootDir().c_str());
    return {FSEC_NOTSUP, std::shared_ptr<FileWriter>()};
}

// 获取子目录：返回的仍然是 LinkDirectory，保持只读属性的传递
FSResult<std::shared_ptr<IDirectory>> LinkDirectory::GetDirectory(const std::string& dirPath) noexcept
{
    std::string absPath = util::PathUtil::JoinPath(GetRootDir(), dirPath);
    // ... 检查路径存在且为目录 ...
    return {FSEC_OK, std::make_shared<LinkDirectory>(absPath, GetFileSystem())};
}
```

代码非常直观地体现了 `LinkDirectory` 的只读特性。通过重写并禁用写操作接口，它为上层应用提供了一个安全、可靠的只读数据视图。

## 4. 结论

`FenceDirectory` 和 `LinkDirectory` 是 Indexlib 文件系统装饰器模式的绝佳范例。它们没有重新发明轮子，而是在现有的 `IDirectory` 接口之上，通过包装和拦截，优雅地增加了新的功能和约束。`FenceDirectory` 通过“临时写入+原子提交”的机制，为分布式环境下的数据写入提供了强大的安全保障，是保证数据一致性的基石。`LinkDirectory` 则通过模拟符号链接和强制只读，为不可变数据提供了一种高效、安全的访问方式。这两个组件极大地增强了 Indexlib 文件系统的能力，使其能够从容应对在线服务、实时索引等复杂场景下的挑战。
