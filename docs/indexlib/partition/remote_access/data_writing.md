
# Indexlib 远程访问：补丁数据写入 (Patch Data Writing)

**涉及文件:**
* `indexlib/partition/remote_access/attribute_patch_data_writer.h`
* `indexlib/partition/remote_access/attribute_patch_data_writer.cpp`
* `indexlib/partition/remote_access/single_value_patch_data_writer.h`
* `indexlib/partition/remote_access/var_num_patch_data_writer.h`
* `indexlib/partition/remote_access/var_num_patch_data_writer.cpp`

## 1. 功能概述

该模块位于 `Data Patching` 流程的最底层，直接负责将增补的属性数据（Attribute Patch Data）持久化到文件系统中。它从 `AttributeDataPatcher` 接收编码后的二进制数据，并根据属性的类型（定长单值 vs. 变长/多值）和配置，将其写入为一个或多个符合 `Indexlib` 存储格式的文件。

此模块的核心职责是：**以高效、正确的方式，将内存中的属性补丁数据序列化到磁盘**。

具体来说，它实现了以下关键功能：

*   **抽象写入接口**：定义了统一的 `AttributePatchDataWriter` 抽象基类，为上层 `Patcher` 提供了 `AppendValue` 和 `AppendNullValue` 等通用接口，屏蔽了底层文件格式的差异。
*   **定长数据写入**：通过 `SingleValuePatchDataWriter`，为定长的单值属性（如 `int32`, `double` 等）提供高效的写入方案。这包括对数据进行可选的等值压缩（Equal Value Compress）。
*   **变长数据写入**：通过 `VarNumPatchDataWriter`，为变长属性（如 `string`）或多值属性（如 `MultiInt32`）提供写入支持。这涉及到同时写入数据文件（`.data`）和偏移量文件（`.offset`），并能处理数据去重和自适应偏移量压缩等高级功能。
*   **元数据生成**：在写入数据的同时，记录并持久化必要的元数据，如总文档数、最长数据项长度等，这些信息在后续读取时至关重要。

## 2. 系统架构与设计

该模块是一个典型的策略模式应用。`AttributePatchDataWriter` 作为抽象的策略接口，`SingleValuePatchDataWriter` 和 `VarNumPatchDataWriter` 作为两种具体的实现策略。上层的 `AttributeDataPatcher` 根据 `AttributeConfig` 来决定在运行时使用哪种策略。

### 2.1. `AttributePatchDataWriter`：抽象的写入策略

这是一个纯虚基类，定义了所有属性补丁写入器必须遵守的契约。

#### 设计动机

*   **统一接口**：为上层模块提供一个稳定、统一的编程接口。上层 `Patcher` 无需关心正在处理的属性是定长的还是变长的，只需调用 `AppendValue` 即可。
*   **隔离变化**：将数据写入的实现细节与数据生成的逻辑（在 `Patcher` 中）分离开。如果未来需要支持新的属性存储格式（例如，一种新的压缩算法），只需增加一个新的 `AttributePatchDataWriter` 子类，而无需修改上层代码。

#### 核心接口

*   `Init(...)`: 初始化 `Writer`，通常会在此方法中创建所需的文件句柄。
*   `AppendValue(const autil::StringView& value)`: 追加一个编码后的非 `Null` 值。
*   `AppendNullValue()`: 追加一个 `Null` 值。
*   `Close()`: 关闭 `Writer`，确保所有缓冲数据被刷盘，并关闭文件句柄。

### 2.2. `SingleValuePatchDataWriter<T>`：定长单值属性写入器

这个类专门用于处理定长的、单值的属性，例如 `int8`, `int32`, `float`, `double` 等。它是通过 C++ 模板实现的，以保证类型安全和高性能。

#### 设计动机

*   **性能优化**：定长数据的处理可以非常高效。数据可以直接以二进制形式追加到文件中，无需存储额外的长度信息。`SingleValuePatchDataWriter` 正是利用了这一点。
*   **压缩支持**：`Indexlib` 对定长数据支持一种名为“等值压缩”（Equal Value Compress）的优化。如果连续的多个文档具有相同的属性值，`Indexlib` 只会存储一次该值，并记录其连续出现的次数。`SingleValuePatchDataWriter` 集成了 `EqualValueCompressDumper` 来支持这种压缩，从而有效减少存储空间。

#### 核心逻辑与实现

1.  **初始化 (`Init`)**：
    *   根据 `AttributeConfig` 判断是否需要数据压缩 (`AttributeCompressInfo::NeedCompressData`)。如果需要，则创建一个 `EqualValueCompressDumper` 实例。
    *   创建一个 `SingleValueAttributeFormatter<T>`，它封装了数据压缩类型和 `Null` 值支持的信息。
    *   在目标目录下创建 `ATTRIBUTE_DATA_FILE_NAME`（通常是 `data`）文件，并获取一个 `FileWriter`。
    *   创建一个 `SingleValueDataAppender`，它是一个数据缓冲器，负责将数据暂存起来，批量写入文件。

2.  **数据追加 (`AppendValue`, `AppendNullValue`)**：
    *   `AppendValue` 接收一个 `StringView`，它首先会校验其长度是否正好等于 `sizeof(T)`，确保数据类型的正确性。
    *   然后，它将 `StringView` 中的数据转换为类型 `T` 的值，并调用 `mDataAppender->Append(value, false)` 将其添加到缓冲区中。
    *   `AppendNullValue` 则直接调用 `mDataAppender->Append(T(), true)`，传入一个默认构造的 `T` 值和一个 `is_null` 标志。
    *   每次 `Append` 后，都会检查缓冲区是否已满 (`mDataAppender->BufferFull()`)。如果已满，则调用 `FlushDataBuffer` 将数据刷盘。

3.  **数据刷盘 (`FlushDataBuffer`)**：
    *   这是数据写入的核心。它检查 `DataAppender` 的缓冲区中是否有数据。
    *   如果启用了压缩（`mCompressDumper` 存在），则调用 `mDataAppender->FlushCompressBuffer(mCompressDumper.get())`。`DataAppender` 会将缓冲区中的数据交给 `CompressDumper` 进行处理。`CompressDumper` 会在内部进行等值压缩，并将压缩后的结果写入它所持有的 `FileWriter`（最终也是写入 `data` 文件）。
    *   如果未启用压缩，则直接调用 `mDataAppender->Flush()`，数据会未经压缩地直接写入文件。

4.  **关闭 (`Close`)**：
    *   首先调用 `FlushDataBuffer` 确保所有剩余在缓冲区的数据被处理。
    *   如果启用了压缩，需要额外调用 `mCompressDumper->Dump(mDataFile)`，以确保压缩器内部剩余的、未形成完整数据块的数据也被写入文件。
    *   最后，关闭文件句柄 (`mDataFile->Close()`)。

    ```cpp
    // indexlib/partition/remote_access/single_value_patch_data_writer.h
    template <typename T>
    inline void SingleValuePatchDataWriter<T>::FlushDataBuffer()
    {
        if (!mDataAppender || mDataAppender->GetInBufferCount() == 0) {
            return;
        }

        if (mCompressDumper) {
            mDataAppender->FlushCompressBuffer(mCompressDumper.get());
        } else {
            assert(mDataFile);
            mDataAppender->Flush();
        }
    }
    ```

### 2.3. `VarNumPatchDataWriter`：变长/多值属性写入器

这个类用于处理所有非定长单值的情况，包括变长类型（如 `string`, `MultiChar`）和多值类型（如 `MultiInt32`）。

#### 设计动机

*   **统一处理变长数据**：所有变长数据的存储都需要一个共同的模式：一个数据文件（`.data`）用于存储紧凑的二进制数据，一个偏移量文件（`.offset`）用于记录每个文档对应的数据在 `.data` 文件中的结束位置。`VarNumPatchDataWriter` 封装了这种通用的“数据+偏移”写入模式。
*   **复用通用组件**：`Indexlib` 内部有一个非常通用的 `VarLenDataWriter` 组件，它已经处理了自适应偏移量压缩、数据去重等复杂逻辑。`VarNumPatchDataWriter` 的设计核心就是对这个通用组件的直接复用，避免了重复造轮子。

#### 核心逻辑与实现

1.  **初始化 (`Init`)**：
    *   在目标目录下创建三个文件：`ATTRIBUTE_DATA_FILE_NAME` (`data`)、`ATTRIBUTE_OFFSET_FILE_NAME` (`offset`) 和 `ATTRIBUTE_DATA_INFO_FILE_NAME` (`data_info`)。
    *   配置一个 `VarLenDataParam` 结构体，将 `AttributeConfig` 中的相关配置（如是否需要压缩偏移量、是否开启自适应偏移量、是否唯一编码等）转换成 `VarLenDataWriter` 所需的参数。
    *   创建一个 `VarLenDataWriter` 实例，并将 `data` 和 `offset` 文件的 `FileWriter` 传递给它进行初始化。
    *   创建一个 `AttributeConvertor` 用于解码从上层传来的 `StringView`。

2.  **数据追加 (`AppendValue`, `AppendNullValue`)**：
    *   `AppendValue` 接收一个编码后的 `StringView`。
    *   它首先调用 `mAttrConvertor->Decode()`，这个方法不仅会返回原始的数据 `data`，还会返回一个 `hashKey`（如果配置了唯一编码 `uniqEncode`）。
    *   然后，它直接调用 `mDataWriter->AppendValue(meta.data, meta.hashKey)`。`VarLenDataWriter` 内部会自动处理以下逻辑：
        *   将 `meta.data` 写入 `data` 文件。
        *   计算新的偏移量，并将其（可能经过压缩）写入 `offset` 文件。
        *   如果开启了去重，会根据 `hashKey` 对数据进行去重。
    *   `AppendNullValue` 会先获取一个表示 `Null` 的特殊编码字符串，然后调用 `AppendValue` 将其写入。

3.  **关闭 (`Close`)**：
    *   调用 `mDataWriter->Close()`，这会确保 `VarLenDataWriter` 内部的所有缓冲（数据和偏移量）都被刷入对应的文件，并关闭文件句柄。
    *   创建一个 `AttributeDataInfo` 对象，它包含了数据项的总数（即文档总数）和数据项的最大长度。这些信息对于读取时初始化 `AttributeReader` 至关重要。
    *   将 `AttributeDataInfo` 对象序列化为字符串，并写入 `data_info` 文件。
    *   关闭 `data_info` 文件的句柄。

    ```cpp
    // indexlib/partition/remote_access/var_num_patch_data_writer.cpp
    void VarNumPatchDataWriter::Close()
    {
        assert(mDataWriter);
        mDataWriter->Close();

        AttributeDataInfo dataInfo(mDataWriter->GetDataItemCount(), mDataWriter->GetMaxItemLength());
        string content = dataInfo.ToString();
        mDataInfoFile->Write(content.c_str(), content.length()).GetOrThrow();
        mDataInfoFile->Close().GetOrThrow();
    }
    ```

## 4. 技术风险与考量

*   **文件IO性能**：作为直接和文件系统打交道的模块，其性能直接受 IO 影响。`MergeIOConfig` 中提供的 `writeBufferSize` 和 `enableAsyncWrite` 是关键的调优参数。合理的缓冲区大小可以减少 `write` 系统调用的次数，异步写入则可以将 IO 操作与计算逻辑解耦，提升总体吞吐量。
*   **数据格式的正确性**：该模块是保证 `Indexlib` 属性文件格式正确的最后一道防线。任何在数据编码、压缩、偏移量计算上的错误，都会导致生成损坏的、无法读取的索引文件。因此，该模块对 `Indexlib` 内部各组件（如 `Formatter`, `Dumper`, `VarLenDataWriter`）的正确使用至关重要。
*   **资源管理**：`Writer` 类在其生命周期内持有文件句柄。`Close` 方法必须被正确调用，否则会导致文件句柄泄露和数据不完整。上层 `Patcher` 的 `Close` 方法保证了这一点。

## 5. 总结

`Patch Data Writing` 模块是 `Indexlib` 数据修改功能的坚实基础。它通过策略模式，为不同类型的属性数据提供了专门优化的高度封装的写入器。`SingleValuePatchDataWriter` 通过模板和对定长数据结构的直接操作实现了极致的性能，并支持等值压缩。`VarNumPatchDataWriter` 则通过复用强大的 `VarLenDataWriter` 组件，优雅地处理了所有变长数据的复杂写入逻辑。这个模块的设计清晰、高效且可扩展，有力地保障了 `Indexlib` 在进行 `Schema` 演化时，能够正确、高效地生成补丁数据。
