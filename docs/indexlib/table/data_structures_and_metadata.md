
# Indexlib Table 数据结构与元数据

**涉及文件:**
* `indexlib/table/segment_meta.h`
* `indexlib/table/segment_meta.cpp`
* `indexlib/table/building_segment_reader.h`
* `indexlib/table/building_segment_reader.cpp`
* `indexlib/table/table_common.h`
* `indexlib/table/table_common.cpp`

## 1. 系统概述

在 Indexlib 的世界里，所有的数据和索引，最终都被组织成一个个的 **Segment**。Segment 是 Indexlib 中数据划分、增量构建和版本管理的基本单元。为了有效地管理和使用这些 Segment，Indexlib 设计了一系列的数据结构和元数据，用于描述 Segment 的属性、状态和生命周期。

本篇文档将深入分析 `table` 模块中与核心数据结构及元数据相关的代码，重点阐述 `SegmentMeta`、`BuildingSegmentReader` 以及一些通用的数据结构，揭示它们在 Indexlib 的数据管理体系中所扮演的重要角色。

## 2. `SegmentMeta`：Segment 的“身份证”

`SegmentMeta` 是一个核心的元数据结构，它描述了一个**已构建完成（Built）**的 Segment 的所有关键信息。你可以将 `SegmentMeta` 看作是 Segment 的“身份证”，它为上层的 `TableReader`、`MergePolicy` 等组件提供了访问和操作 Segment 所需的全部上下文。

### 关键设计与实现

`SegmentMeta` 主要由以下几个部分组成：

*   **`segmentDataBase` (`index_base::SegmentDataBase`)**: 这是 `SegmentMeta` 中最重要的部分之一。它存储了 Segment 的基础信息，例如 `segmentId`、`docCount`、`baseDocId` 等。这些信息是定位和访问 Segment 中文档的基础。

*   **`segmentInfo` (`index_base::SegmentInfo`)**: `segmentInfo` 存储了 Segment 的一些统计和状态信息，例如 Segment 的总大小、时间戳、`locator` 等。这些信息在合并策略、版本管理和增量恢复等场景中，起着至关重要的作用。

*   **`segmentDataDirectory` (`file_system::DirectoryPtr`)**: 这个字段是一个指向 Segment 数据所在目录的智能指针。通过这个 `Directory` 对象，上层组件可以访问 Segment 的所有物理文件，例如索引文件、属性文件等。

*   **`type` (`SegmentSourceType`)**: 这是一个枚举类型，用于标识 Segment 的来源。它主要有两个值：
    *   `INC_BUILD`: 表示该 Segment 是由离线或增量构建生成的。
    *   `RT_BUILD`: 表示该 Segment 是由实时构建生成的。

通过 `SegmentMeta`，Indexlib 将一个 Segment 的逻辑元数据和物理数据访问接口，有机地组织在了一起，为上层模块提供了一个清晰、统一的 Segment 视图。

#### 核心代码示例 (`segment_meta.h`)

```cpp
class SegmentMeta
{
public:
    enum class SegmentSourceType { INC_BUILD = 0, RT_BUILD = 1, UNKNOWN = -1 };

public:
    SegmentMeta();
    ~SegmentMeta();

public:
    index_base::SegmentDataBase segmentDataBase;
    index_base::SegmentInfo segmentInfo;
    file_system::DirectoryPtr segmentDataDirectory;
    SegmentSourceType type;
};
```

## 3. `BuildingSegmentReader`：构建中 Segment 的“快照”

与 `SegmentMeta` 描述已完成的 Segment 不同，`BuildingSegmentReader` 描述的是一个**正在构建中（Building）**的 Segment。当 `TableWriter` 在内存中构建索引时，它会对外提供一个 `BuildingSegmentReader`，使得查询可以**实时**地访问到正在写入的数据。

### 关键设计与实现

`BuildingSegmentReader` 的设计相对简单，它主要包含以下信息：

*   **`_buildingSegId`**: 正在构建的 Segment 的 ID。
*   **`_dependentSegIds`**: 一个 `set`，用于记录该 `Building Segment` 依赖的其他 Segment 的 ID。这种情况通常发生在实时构建场景中，例如，一个实时 Segment 可能需要依赖上一个实时 Segment 的某些数据（如主键信息）。

`BuildingSegmentReader` 是一个基类，它只定义了最基本的信息。具体的 Table 类型需要继承 `BuildingSegmentReader`，并实现自己的实时读取逻辑。例如，一个 KV Table 的 `BuildingSegmentReader` 子类，需要能够从内存中的 `HashMap` 中读取数据。

通过 `BuildingSegmentReader`，Indexlib 实现了**近实时（Near Real-Time）**的搜索能力，即数据在写入后，几乎可以立即被检索到。

#### 核心代码示例 (`building_segment_reader.h`)

```cpp
class BuildingSegmentReader
{
public:
    BuildingSegmentReader(segmentid_t buildingSegId);
    virtual ~BuildingSegmentReader();

public:
    segmentid_t GetSegmentId() const { return _buildingSegId; }
    const std::set<segmentid_t>& GetDependentSegmentIds() const { return _dependentSegIds; }
    void AddDependentSegmentId(segmentid_t segId) { _dependentSegIds.insert(segId); }

protected:
    segmentid_t _buildingSegId;
    std::set<segmentid_t> _dependentSegIds;
};
```

## 4. `table_common.h`：通用的数据结构

`table_common.h` 文件中定义了一些在 `table` 模块的各个组件之间传递信息时，会用到的通用数据结构。这些结构体通常以 `...Description` 或 `...Param` 的形式命名，体现了**参数对象（Parameter Object）**的设计模式。

*   **`BuildSegmentDescription`**: 当 `TableWriter` 完成一个 Segment 的构建并将其转储（Dump）到磁盘时，它会填充这个结构体，向上层返回新生成 Segment 的一些描述信息，例如 `segmentStats`（Segment 的统计信息）、`deployFileList`（需要发布的文件列表）等。

*   **`MergeSegmentDescription`**: 与 `BuildSegmentDescription` 类似，这个结构体用于描述一个**合并后**生成的新 Segment 的信息。它由 `MergePolicy` 在 `CreateMergeTaskDescriptions` 方法中填充。

*   **`TableWriterInitParam`**: 这个结构体在 `TableWriter` 的初始化（`Init`）时使用。它将 `TableWriter` 所需的所有依赖，如 `Schema`、`Options`、`TableResource`、文件系统目录等，都封装在了一起。这种设计使得 `TableWriter` 的 `Init` 接口更加简洁、稳定，即使未来需要增加新的依赖，也无需修改接口的签名。

使用参数对象来传递依赖，是 Indexlib 中一种常见的设计模式。它不仅可以提高代码的可读性和可维护性，还能有效地将组件的创建和其依赖的获取分离开来，降低了模块间的耦合度。

## 5. 总结

Indexlib 的**数据结构与元数据**设计，是其整个数据管理体系的基石。`SegmentMeta` 为已完成的 Segment 提供了全面的描述，`BuildingSegmentReader` 为正在构建的 Segment 提供了实时的视图，而 `table_common.h` 中的一系列通用数据结构，则保证了各个组件之间信息传递的顺畅和高效。这些精心设计的数据结构，共同构建了一个清晰、一致、可扩展的数据模型，为 Indexlib 的稳定性、性能和灵活性提供了有力的支撑。
