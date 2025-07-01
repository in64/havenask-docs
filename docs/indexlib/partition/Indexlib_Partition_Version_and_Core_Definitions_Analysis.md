
# Indexlib 分区版本与核心定义解析

**涉及文件:**
* `indexlib/partition/partition_version.h`
* `indexlib/partition/partition_version.cpp`
* `indexlib/partition/partition_define.h`

## 1. 概述

在 Indexlib 的世界里，数据并非一成不变，而是随着时间的推移不断地被添加、更新和删除。为了有效地管理这些变化，并为上层查询提供一个稳定、一致的数据视图，Indexlib 引入了“版本”的概念。本文将深入探讨 Indexlib 中与分区版本管理密切相关的核心类 `PartitionVersion`，以及在分区层级定义的一些关键数据结构，揭示 Indexlib 是如何通过这些基础组件来追踪和管理索引的演进过程的。

`PartitionVersion` 可以看作是分区在某个特定时间点的“快照”，它精确地描述了该时间点上，哪些数据是“在线”可查的。而 `partition_define.h` 中定义的一系列数据结构，则为在分区层级组织和传递信息提供了统一的规范。理解这些核心定义，是深入探索 Indexlib 更高层功能（如索引读写、查询处理）的基础。

## 2. `PartitionVersion`：分区演进的“快照”

`PartitionVersion` 是一个至关重要的类，它封装了一个分区在某个特定时间点的完整版本信息。这个版本信息不仅仅是一个简单的版本号，而是由多个维度的信息共同构成的。

### 2.1. 核心构成

`PartitionVersion` 主要由以下几个部分组成：

*   **`index_base::Version` (`mVersion`):** 这是 `PartitionVersion` 的核心。`index_base::Version` 对象本身就包含了丰富的信息，最主要的是一个 segment ID 的列表，这个列表定义了在该版本中，所有已经构建完成并且“在线”的 segment。Segment 是 Indexlib 中索引数据的基本组成单元。
*   **`segmentid_t mBuildingSegmentId`:** 表示当前正在构建中的 segment 的 ID。这个 segment 中的数据是实时写入的，但尚未固化成一个完整的、不可变的 segment。
*   **`segmentid_t mLastLinkRtSegId`:** 表示最后一个链接的实时 segment 的 ID。这在某些特定的实时场景下（如本地存储）会用到。
*   **`std::set<segmentid_t> mDumpingSegmentIds`:** 表示当前正在从内存 dump 到磁盘的 segment 的 ID 集合。这些 segment 处于一个中间状态，即将完成构建。

通过将这几部分信息组合在一起，`PartitionVersion` 能够精确地描述出在任何一个时间点，一个分区中所有 segment 的状态（已上线、正在构建、正在 dump）。

### 2.2. 关键功能

#### 2.2.1. 判断 Segment 的使用状态

`IsUsingSegment` 方法可以用来判断一个给定的 segment ID 在当前版本中是否处于“使用中”的状态。一个 segment 被认为是“使用中”，如果它满足以下任一条件：

*   它存在于 `mVersion` 的 segment 列表中（即已上线）。
*   它等于 `mBuildingSegmentId`（即正在构建中）。
*   它存在于 `mDumpingSegmentIds` 集合中（即正在 dump 中）。

```cpp
// indexlib/partition/partition_version.cpp

bool PartitionVersion::IsUsingSegment(segmentid_t segId, TableType type) const
{
    if (mVersion.HasSegment(segId)) {
        return true;
    }

    if (mDumpingSegmentIds.find(segId) != mDumpingSegmentIds.end()) {
        return true;
    }

    return mBuildingSegmentId == segId;
}
```

这个函数对于索引的清理和回收至关重要。只有当一个 segment 在所有现存的 `PartitionVersion` 中都不再被使用时，它对应的物理文件才能被安全地删除。

#### 2.2.2. 生成版本哈希 ID

`GetHashId` 方法用于为当前的 `PartitionVersion` 生成一个唯一的 64 位哈希值。这个哈希值是对版本信息的紧凑表示，可以用于快速地比较两个版本是否相同。

```cpp
// indexlib/partition/partition_version.cpp

uint64_t PartitionVersion::GetHashId() const
{
    string str =
        mVersion.ToString() + StringUtil::toString(mBuildingSegmentId) + StringUtil::toString(mLastLinkRtSegId);

    uint64_t hashKey = 0;
    util::MurmurHasher::GetHashKey(str.c_str(), str.size(), hashKey);
    return hashKey;
}
```

值得注意的是，`GetHashId` 的计算并没有包含 `mDumpingSegmentIds`。这可能是一个设计上的考量，因为 dump 中的 segment 状态变化非常快，如果将其包含在哈希计算中，会导致版本哈希值频繁变动。

## 3. `partition_define.h`：分区层的核心定义

`partition_define.h` 文件中定义了一系列在分区层级广泛使用的数据结构和枚举类型。这些定义为不同模块之间的信息交换提供了统一的“语言”。

### 3.1. `IndexPartitionReaderType`

这是一个枚举类型，用于标识 `IndexPartitionReader` 的类型。在主子表结构中，一个分区可能既有主表读句柄，也有子表读句柄。这个枚举就是用来区分它们的。

```cpp
// indexlib/partition/partition_define.h

enum IndexPartitionReaderType {
    READER_UNKNOWN_TYPE,
    READER_PRIMARY_MAIN_TYPE, // 主表主 part
    READER_PRIMARY_SUB_TYPE,  // 主表 sub part
};
```

### 3.2. `IndexPartitionReaderInfo` 和 `TabletReaderInfo`

这两个结构体用于封装一个读句柄及其相关的元信息。`IndexPartitionReaderInfo` 用于传统的 `IndexPartition` 架构，而 `TabletReaderInfo` 用于新的 `Tablet` 架构。

```cpp
// indexlib/partition/partition_define.h

struct IndexPartitionReaderInfo {
    IndexPartitionReaderType readerType;
    IndexPartitionReaderPtr indexPartReader;
    std::string tableName;
    uint64_t partitionIdentifier;
};

struct TabletReaderInfo {
    IndexPartitionReaderType readerType;
    std::shared_ptr<indexlibv2::framework::ITabletReader> tabletReader;
    std::string tableName;
    uint64_t partitionIdentifier;
    bool isLeader = false;
    const indexlibv2::framework::TabletInfos* tabletInfos = nullptr;
};
```

这些结构体通常在 `IndexApplication` 创建快照时被使用，用于将所有分区的读句柄信息收集起来，传递给 `PartitionReaderSnapshot`。

### 3.3. `TableMem2IdMap`

这是一个非常实用的辅助类，它实现了一个从 `(表名, 成员名)` 到 `ID` 的映射。这里的“成员”可以是一个索引字段（index）、一个属性字段（attribute）或者一个 pack attribute。

```cpp
// indexlib/partition/partition_define.h

class TableMem2IdMap
{
public:
    typedef std::map<std::pair<std::string, std::string>, uint32_t> IdentityMap;
    // ...
    bool Find(const std::string& tableName, const std::string& id, uint32_t& value) const;
    void Insert(const std::string& tableName, const std::string& id, uint32_t value);
    // ...
private:
    IdentityMap mIdMap;
};
```

在 `IndexApplication` 中，会为索引、属性和 pack attribute 分别维护一个 `TableMem2IdMap`。这使得在查询时，可以根据表名和字段名，快速地定位到对应的内部 ID，进而找到相应的读句柄。

## 4. 总结

`PartitionVersion` 和 `partition_define.h` 中的核心定义，共同构成了 Indexlib 分区管理的基石。`PartitionVersion` 通过对不同状态的 segment 进行精细化的描述，为 Indexlib 提供了强大的版本管理能力，这是实现索引数据一致性、支持增量构建和近实时查询的关键。而 `partition_define.h` 中的各种数据结构，则通过标准化的信息封装，保证了分区层级各个模块之间能够高效、准确地协同工作。

对这些基础概念的深入理解，将为我们后续探索 Indexlib 更复杂的功能（如查询处理、数据合并等）扫清障碍，并帮助我们更好地领会 Indexlib 作为一个高性能、可扩展索引系统的设计精髓。
