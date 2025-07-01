
# Indexlib 哈希表核心实现深度解析

**涉及文件:**
*   `index/common/hash_table/DenseHashTable.h`
*   `index/common/hash_table/CuckooHashTable.h`
*   `index/common/hash_table/ChainHashTable.h`
*   `index/common/hash_table/SeparateChainHashTable.h`

## 1. 系统概述

Indexlib 中的哈希表模块是其高性能键值（KV）和主键（Primary Key）索引的核心支撑。为了应对不同场景下对内存使用、读写性能、构建效率的复杂需求，该模块设计并实现了多种哈希表策略。本文档深入剖析了其中最为核心的三种哈希表实现：**密集哈希表（Dense Hash Table）**、**布谷鸟哈希表（Cuckoo Hash Table）** 和 **拉链法哈希表（Chain/Separate Chain Hash Table）**。

这三种实现共同构成了 Indexlib 的哈希表矩阵：

*   **密集哈希表**：采用线性探测的开放寻址法，结构简单，缓存友好，在低冲突率下拥有极致的查找性能。但它对哈希函数的质量和装载因子（Occupancy）非常敏感，高冲突时性能会急剧下降。
*   **布谷鸟哈希表**：一种更现代的开放寻址法，通过使用多个哈希函数和“踢出-重哈希”机制，实现了极高的空间利用率和优秀的读性能。它在写入时可能会有性能抖动（踢出操作），但通常能维持 O(1) 的期望查找时间。
*   **拉链法哈希表**：经典的数据结构实现，通过在每个桶（Bucket）后挂载一个链表来解决哈希冲突。它的写入性能稳定，对装载因子不敏感，但由于链表导致的指针跳转，其缓存局部性较差，读取性能通常逊于前两者。

下文将逐一拆解这些哈希表的架构设计、关键算法、技术权衡以及潜在的风险点。

---

## 2. 密集哈希表 (DenseHashTable)

密集哈希表是基于开放寻址法的一种实现，当发生哈希冲突时，它会探测（Probe）后续的存储桶，直到找到一个空桶为止。Indexlib 中采用的是最简洁的**线性探测（Linear Probing）**策略。

### 2.1. 架构与设计动机

**设计目标**：追求极致的读取性能和内存局部性。

线性探测的核心思想是，当一个键 `key` 哈希到 `bucketId` 位置时，如果该位置已被占用，则依次检查 `bucketId + 1`, `bucketId + 2`, ... 直到找到空桶。这种连续内存访问的方式对 CPU 缓存极为友好，可以有效利用缓存行（Cache Line）预取机制，从而在低冲突率下获得极高的查找效率。

其物理布局为一个连续的 `Bucket` 数组，不涉及任何间接指针，内存结构非常紧凑。这种设计在构建完成后，可以被完整地 dump 到磁盘，并以内存映射（mmap）的方式加载，实现近乎零开销的只读访问。

### 2.2. 核心逻辑与算法

#### 2.2.1. 查找 (Find)

查找过程完美体现了线性探测的机制。

1.  **计算初始桶位**：`bucketId = key % bucketCount`。
2.  **线性探测**：
    *   检查 `_bucket[bucketId]`。
    *   如果桶为空（`IsEmpty()`），则说明 `key` 不存在，查找失败。
    *   如果桶中的键与 `key` 匹配（`IsEqual(key)`），则查找成功。根据桶的状态（`IsDeleted()`）返回相应结果。
    *   如果桶被其他键占用，则增加探测次数 `probeCount`，计算下一个桶位 `bucketId = (bucketId + JUMP) % bucketCount`（其中 `JUMP` 宏定义为1，即线性探测），然后重复此过程。
3.  **终止条件**：为了防止无限循环（在表满的情况下），探测次数 `probeCount` 不能超过总桶数 `bucketCount`。如果超过，则认为表结构异常。

**关键代码 (`InternalFindBucket`)**:
```cpp
template <typename _KT, typename _VT, bool HasSpecialKey, bool useCompactBucket>
inline typename DenseHashTableBase<_KT, _VT, HasSpecialKey, useCompactBucket>::Bucket*
DenseHashTableBase<_KT, _VT, HasSpecialKey, useCompactBucket>::InternalFindBucket(const _KT& key) const
{
    uint64_t bucketCount = _bucketCount;
    uint64_t bucketId = key % bucketCount;
    uint64_t probeCount = 0;
    while (true) {
        Bucket& bucket = _bucket[bucketId];
        if (bucket.IsEmpty()        // not found
            || bucket.IsEqual(key)) // found it or deleted
        {
            return &bucket;
        }
        if (unlikely(!Probe(key, probeCount, bucketId, bucketCount))) {
            return NULL;
        }
    }
    // ... a few lines of statistics logging ...
    return NULL;
}

template <typename _KT, typename _VT, bool HasSpecialKey, bool useCompactBucket>
inline bool DenseHashTableBase<_KT, _VT, HasSpecialKey, useCompactBucket>::Probe(const _KT& key, uint64_t& probeCount,
                                                                                 uint64_t& bucketId,
                                                                                 uint64_t bucketCount)
{
    ++probeCount;
    bucketId += DENSE_HASH_TABLE_JUMP(probeCount); // DENSE_HASH_TABLE_JUMP is 1
    if (unlikely(bucketId >= bucketCount)) {
        bucketId %= bucketCount;
    }
    if (unlikely(probeCount >= bucketCount)) {
        AUTIL_LOG(ERROR, "too many probings for key[%lu], probeCount[%lu], bucketCount[%lu]",
                  (uint64_t)key, probeCount, bucketCount);
        return false;
    }
    return true;
}
```

#### 2.2.2. 插入 (Insert)

插入操作复用了 `InternalFindBucket` 的逻辑。

1.  调用 `InternalFindBucket(key)` 找到目标桶。
2.  如果返回的桶是空桶（`IsEmpty()`），说明这是一个新键，直接在该桶中设置键值对，并增加 `keyCount`。
3.  如果返回的桶中已存在该键（`IsEqual(key)`），则更新其值。如果原桶是“已删除”状态，则需要将删除计数器 `_deleteCount` 减一。

#### 2.2.3. 删除 (Delete)

Indexlib 的 KV 场景要求支持逻辑删除。`DenseHashTable` 通过在 `Bucket` 中设置一个特殊状态来实现。

1.  调用 `InternalFindBucket(key)` 找到目标桶。
2.  如果桶存在且未被标记为删除，则调用 `bucket.SetDelete()`，并增加 `_deleteCount`。
3.  **注意**：不能直接将桶设置为空。因为这会破坏线性探测链，导致后续的元素无法被找到。例如，A、B 都哈希到位置 5，A 先插入，B 插入到位置 6。如果此时删除 A 并将位置 5 置空，那么查找 B 时，在位置 5 就会提前终止，导致 B 丢失。

### 2.3. 技术风险与权衡

*   **聚集（Clustering）问题**：线性探测的最大弊端。连续的占用桶会形成“聚集块”，新的插入会使得这些块增长，导致后续插入和查找的探测长度越来越长，性能严重下降。
*   **对装载因子敏感**：当装载因子（`keyCount / bucketCount`）超过一定阈值（如 70-80%），性能会急剧恶化。因此，`DenseHashTable` 默认的 `OCCUPANCY_PCT` 仅为 50%，这是一种用空间换时间的策略，牺牲了内存密度来保证性能。
*   **哈希函数质量**：一个分布不均匀的哈希函数会加剧聚集问题，使得性能远低于理论预期。

---

## 3. 布谷鸟哈希表 (CuckooHashTable)

布谷鸟哈希表是另一种高性能的开放寻址法。它的名字来源于布谷鸟将自己的蛋产在其他鸟巢，并可能将原来的蛋踢出的行为。

### 3.1. 架构与设计动机

**设计目标**：在维持 O(1) 期望查找时间的同时，实现比密集哈希表更高的空间利用率。

其核心思想是：

1.  **多哈希函数**：为每个键 `key` 计算多个（通常是 2 个或更多）独立的哈希值，对应多个候选桶位。
2.  **查找**：查找时，检查所有候选桶位。只要有一个命中，即可返回。这使得查找非常快。
3.  **插入与踢出**：插入新键时，检查其所有候选桶位。
    *   如果有空桶，直接放入。
    *   如果所有候选桶都已占用，则随机选择一个桶，将原有的键“踢出”，然后为被踢出的键寻找它的下一个候选位置。
    *   这个踢出过程可能会像多米诺骨牌一样持续下去，直到找到一个空桶，或者达到预设的踢出次数上限。

Indexlib 的实现还引入了 **分块（Blocking）** 的概念，将 `BLOCK_SIZE` (通常为4) 个桶组成一个块。一个键的哈希值会定位到一个块，然后在块内进行查找和放置。这进一步增强了缓存局部性。

### 3.2. 核心逻辑与算法

#### 3.2.1. 查找 (Find)

查找过程非常直接：

1.  对给定的 `key`，计算 `_nu_hashFunc` 个哈希值。
2.  每个哈希值通过 `GetFirstBucketIdInBlock` 映射到一个块的起始 `bucketId`。
3.  遍历该块内的所有 `BLOCK_SIZE` 个桶，检查是否有键匹配。
4.  对所有哈希函数重复此过程，直到找到匹配的键或检查完所有候选位置。

**关键代码 (`FindBucketForRead`)**:
```cpp
template <typename _KT, typename _VT, bool HasSpecialKey, bool useCompactBucket>
inline const typename CuckooHashTableBase<_KT, _VT, HasSpecialKey, useCompactBucket>::Bucket*
CuckooHashTableBase<_KT, _VT, HasSpecialKey, useCompactBucket>::FindBucketForRead(const _KT& key,
                                                                                  uint8_t nu_hashFunc) const
{
    // Prefetching for the first two hash locations
    uint64_t bucketId = GetFirstBucketIdInBlock(CuckooHash(key, 0), _blockCount);
    __builtin_prefetch(&(_bucket[bucketId]), 0, 1);
    const _KT& hash = CuckooHash(key, 1);
    uint64_t nextId = GetFirstBucketIdInBlock(hash, _blockCount);
    __builtin_prefetch(&(_bucket[nextId]), 0, 1);

    // Check the first block
    for (uint32_t inBlockId = 0; inBlockId < BLOCK_SIZE; ++inBlockId) {
        const Bucket& curBucket = _bucket[bucketId + inBlockId];
        if (curBucket.IsEmpty()) { return NULL; } 
        else if (curBucket.IsEqual(key)) { return &curBucket; }
    }
    // Check the second block
    for (uint32_t inBlockId = 0; inBlockId < BLOCK_SIZE; ++inBlockId) {
        const Bucket& curBucket = _bucket[nextId + inBlockId];
        if (curBucket.IsEmpty()) { return NULL; }
        else if (curBucket.IsEqual(key)) { return &curBucket; }
    }
    // Check remaining hash locations
    for (uint32_t hashFuncId = 2; hashFuncId < nu_hashFunc; ++hashFuncId) {
        // ... similar logic ...
    }
    return NULL;
}
```
代码中使用了 `__builtin_prefetch` 来提前加载数据到缓存，这是一个非常有效的性能优化手段。

#### 3.2.2. 插入与踢出 (Insert & CuckooKick)

插入是布谷鸟哈希表最复杂的部分。

1.  **寻找空位**：首先像查找一样，检查所有候选块。如果任何一个块内有空桶，则直接插入，操作完成。
2.  **启动踢出**：如果所有候选块都满了，就需要启动踢出流程。这个流程通过一个广度优先搜索（BFS）来实现，寻找一条从当前键的某个候选位置开始，最终能到达一个空桶的“踢出路径”。
3.  **BFS寻找路径 (`BFSFindBucket`)**：
    *   将待插入键的所有候选块作为 BFS 的第一层节点。
    *   从队列中取出一个块，遍历其中的每个桶。对于桶中的键 `k_old`，计算它的所有候选块。
    *   如果 `k_old` 的某个候选块 `B_new` 尚未在本次 BFS 中被访问过，则将 `B_new` 加入队列，并记录路径（`B_new` 是由 `k_old` 从当前块踢出的）。
    *   如果在探索过程中，发现某个块 `B_empty` 中有空桶，则说明找到了一条踢出路径。
4.  **执行踢出 (`CuckooKick`)**：
    *   从找到的空桶 `B_empty` 开始，沿着 BFS 记录的路径反向操作。
    *   将路径上一环的元素移动到当前空位，制造出新的空位。
    *   重复此过程，直到最初待插入键的候选位置出现空位。
    *   将新键插入该空位。
5.  **Rehash**：如果 BFS 搜索了预设的深度（`_bFSDepth`）仍未找到空桶，或者踢出次数过多，说明哈希表现状可能过于拥挤或陷入了循环。此时会触发 **Rehash**：
    *   增加哈希函数数量（`_nu_hashFunc++`），为每个键提供更多选择。
    *   如果增加哈希函数仍无法解决，最终可能需要扩展整个哈希表的大小，并重新插入所有元素。

### 2.3. 技术风险与权衡

*   **写入性能抖动**：虽然期望插入时间是 O(1)，但在最坏情况下，一次插入可能触发一长串的踢出操作，甚至 Rehash，导致性能瞬时下降。这对于延迟敏感的在线服务可能是一个风险。
*   **循环问题**：踢出过程可能形成环路，导致无限循环。实现中必须有最大踢出次数的保护机制来触发 Rehash。
*   **实现复杂性**：相比线性探测，布谷鸟哈希的实现逻辑（特别是 BFS 寻路和 CuckooKick）要复杂得多，给调试和维护带来挑战。
*   **高空间利用率**：其主要优点。布谷鸟哈希表可以在装载因子达到 90% 甚至 95% 以上时，仍然保持良好的性能，内存效率远高于密集哈希表。

---

## 4. 拉链法哈希表 (Chain/SeparateChainHashTable)

这是最传统和经典的哈希表实现，Indexlib 中提供了两种略有不同的变体：`ChainHashTable` 和 `SeparateChainHashTable`。它们都遵循拉链法的基本思想。

### 4.1. 架构与设计动机

**设计目标**：提供一种简单、稳定、对装载因子不敏感的哈希表实现。

拉链法的架构非常直观：

1.  **桶数组（Bucket Array）**：一个连续的数组，每个元素是一个指针（或偏移量），指向一个链表的头节点。
2.  **节点池（Node Pool）**：存储实际键值对（KeyNode）的内存区域。
3.  **哈希冲突处理**：所有哈希到同一个桶索引的键，都会被串在同一个链表上。

这种设计的优点是：
*   **插入稳定**：插入操作永远是 O(1) 的，只需在对应链表的头部插入一个新节点即可。
*   **无聚集问题**：不同哈希值的键之间不会相互影响。
*   **高装载因子**：装载因子可以远大于 100%，因为一个桶可以容纳任意多个元素。

`ChainHashTable` 和 `SeparateChainHashTable` 的主要区别在于内存管理和具体实现细节上，但核心原理一致。`SeparateChainHashTable` 使用了 `MMapVector` 来管理节点，更适合需要动态增长和持久化的场景。

### 4.2. 核心逻辑与算法

#### 4.2.1. 查找 (Find)

1.  **定位桶**：`bucketIdx = key % _bucketCount`。
2.  **获取链表头**：从 `_bucket[bucketIdx]` 获取链表头的偏移量/指针。
3.  **遍历链表**：沿着 `next` 指针遍历链表，逐一比较节点中的键，直到找到匹配的键或到达链表末尾。

**关键代码 (`SeparateChainHashTable::Find`)**:
```cpp
template <typename KeyType, typename ValueType>
inline typename SeparateChainHashTable<KeyType, ValueType>::KeyNode*
SeparateChainHashTable<KeyType, ValueType>::Find(const KeyType& key)
{
    uint32_t bucketIdx = key % _bucketCount;
    uint32_t offset = _bucket[bucketIdx];
    while (offset != INVALID_KEY_OFFSET) {
        KeyNode* keyNode = &(*_keyNodes)[offset];
        if (keyNode->key == key) {
            return keyNode;
        }
        offset = keyNode->next;
    }
    return NULL;
}
```

#### 4.2.2. 插入 (Insert)

1.  **创建新节点**：在节点池中创建一个新的 `KeyNode`，包含要插入的键和值。
2.  **定位桶**：`bucketIdx = key % _bucketCount`。
3.  **头插法**：
    *   将新节点的 `next` 指针指向当前桶的链表头（`_bucket[bucketIdx]`）。
    *   更新桶的头指针，使其指向新创建的节点。

**关键代码 (`SeparateChainHashTable::Insert`)**:
```cpp
template <typename KeyType, typename ValueType>
inline void SeparateChainHashTable<KeyType, ValueType>::Insert(const KeyType& key, const ValueType& value)
{
    assert(Find(key) == NULL);
    uint32_t bucketIdx = key % _bucketCount;
    KeyNode keyNode(key, value, _bucket[bucketIdx]);
    uint32_t keyOffset = _keyNodes->Size();
    _keyNodes->PushBack(keyNode);
    MEMORY_BARRIER();
    _bucket[bucketIdx] = keyOffset;
}
```
代码中 `MEMORY_BARRIER()` 的使用是为了确保在多线程环境下，节点内容的写入先于桶指针的更新，防止其他线程读到不完整的节点数据。

### 4.3. 技术风险与权衡

*   **缓存不友好**：链表的节点在内存中是离散分布的，遍历链表会导致大量的指针跳转和随机内存访问，无法有效利用 CPU 缓存，这是其读取性能通常不如开放寻址法的主要原因。
*   **长链表问题**：如果哈希函数不好，或者桶的数量相对于键的数量太少，可能导致某些链表过长，查找性能退化为 O(n)。
*   **内存开销**：每个节点都需要额外的空间来存储 `next` 指针，存在一定的内存开销。

## 5. 总结与比较

| 特性 | 密集哈希表 (Dense) | 布谷鸟哈希表 (Cuckoo) | 拉链法哈希表 (Chaining) |
| :--- | :--- | :--- | :--- |
| **核心思想** | 开放寻址法（线性探测） | 开放寻址法（多哈希+踢出） | 链表解决冲突 |
| **读取性能** | 极高（低冲突时） | 非常高 | 中等（受缓存影响） |
| **写入性能** | 非常高 | 良好，但可能抖动 | 非常高且稳定 |
| **空间利用率**| 低（需保持低装载因子） | 非常高（可达95%+） | 高（可超过100%） |
| **实现复杂度**| 简单 | 复杂 | 中等 |
| **主要优点** | 缓存友好，读取快 | 空间效率极高，读取快 | 写入稳定，无性能悬崖 |
| **主要缺点** | 聚集问题，对装载率敏感 | 写入可能抖动，实现复杂 | 缓存不友好，长链表风险 |
| **适用场景** | 读多写少、键集合相对固定、对延迟极敏感的场景 | 内存敏感、高密度存储、能容忍偶发写入抖动的场景 | 写密集、键集合动态变化、追求实现简单稳定的场景 |

Indexlib 通过提供这一系列精心设计的哈希表实现，为上层业务提供了灵活且强大的工具集，使其能够根据具体的业务需求和硬件环境，选择最优的数据结构，从而在性能和资源消耗之间取得理想的平衡。
