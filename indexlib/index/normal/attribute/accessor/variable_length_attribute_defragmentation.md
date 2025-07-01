
# Indexlib 变长属性数据碎片整理代码分析

**涉及文件:**

*   `indexlib/index/normal/attribute/accessor/var_num_attribute_defrag_slice_array.cpp`
*   `indexlib/index/normal/attribute/accessor/var_num_attribute_defrag_slice_array.h`

## 1. 功能概述

本模块的核心功能是解决 Indexlib 中变长属性（Variable-Length Attribute）的存储碎片问题。变长属性（如字符串、`MultiValue` 类型的字段）的数据部分通常存储在一个称为 `SliceArray` 的连续内存块中，而每个文档只存储一个指向其实际数据位置的偏移量（Offset）。当文档被删除或更新（更新后的数据长度可能变短）时，原先占用的 `SliceArray` 空间就无法被立即重用，从而产生了“碎片”。

随着不断的更新和删除，这些碎片会越积越多，导致：

1.  **空间浪费:** 大量内存或磁盘空间被无效数据占据。
2.  **性能下降:** 如果不进行整理，`SliceArray` 会持续增长，可能影响缓存效率和IO性能。

`VarNumAttributeDefragSliceArray` 的作用就是实现一种在线的“垃圾回收”和“碎片整理”机制。它能够识别出那些碎片率过高的 `Slice`（`SliceArray` 的基本分配单元），并将其中仍然“存活”的有效数据搬迁到新的 `Slice` 中，最后将旧的、不再有任何有效数据的 `Slice` 整个回收，从而达到节省空间、提高存储效率的目的。

## 2. 系统架构与核心组件

该模块的设计是基于 Indexlib 底层的 `DefragSliceArray` 组件，并针对变长属性的特点进行了特化。其核心思想是经典的“拷贝-回收”（Copy-and-Collect）垃圾回收算法。

<img src="https://g.alicdn.com/imgextra/i2/O1CN01k7g7k71CqQy4g4f8g_!!6000000001484-2-tps-1384-776.png" alt="Defrag Architecture" width="600"/>

**核心组件:**

*   **`VarNumAttributeDefragSliceArray`**:
    *   **功能:** 继承自通用的 `util::DefragSliceArray`，并实现了针对变长属性的碎片整理逻辑。
    *   **核心依赖:**
        *   `AttributeOffsetReader`: 用于读取和更新每个 `docId` 对应的偏移量（Offset）。这是连接文档和其属性数据的桥梁。
        *   `UpdatableVarNumAttributeOffsetFormatter`: 负责偏移量的编码和解码。偏移量中不仅包含了数据在 `SliceArray` 中的位置，还可能包含其他元信息。
        *   `VarNumAttributeDataFormatter`: 负责解析存储在 `SliceArray` 中的原始数据，能从中提取出数据的实际长度。
        *   `AttributeMetrics`: 用于统计和监控碎片整理过程中的各项指标，如回收的空间大小、移动的数据量等。
    *   **核心逻辑:**
        1.  **触发条件 (`NeedDefrag`)**: 碎片整理不是随时都在进行的。该方法会判断一个给定的 `sliceIdx` 是否需要被整理。判断的依据是该 `Slice` 的“浪费空间”（`wastedSize`）是否超过了一个预设的阈值（`mDefragPercentThreshold`）。通常，只有当一个 `Slice` 的大部分空间都变成碎片时，才值得花费 CPU 和 IO 去整理它。正在被写入的当前 `Slice` 不会被整理。
        2.  **碎片整理 (`Defrag`)**: 这是核心执行逻辑。一旦 `NeedDefrag` 返回 `true`，该方法就会被调用。
            *   它会遍历一个 `Slice` 覆盖的偏移量范围 (`offsetBegin` 到 `offsetEnd`)。
            *   然后，它会迭代所有的文档（`docId` 从 0 到 `docCount`），检查每个文档的偏移量是否落在了当前正在整理的 `Slice` 内。
            *   如果落在了范围内，说明这个文档的数据是“存活”的，需要被搬迁。此时会调用 `MoveData`。
            *   当所有文档都检查完毕后，这个旧的 `Slice` 里的所有存活数据都已经被拷贝走了。于是，整个 `Slice` 的 `wastedSize` 被设置为其总长度，意味着它已经变成一个完全无用的 `Slice`。
            *   最后，将这个无用的 `sliceIdx` 加入到 `mUselessSliceIdxs` 列表中，等待后续被统一释放。
        3.  **数据搬迁 (`MoveData`)**: 这是一个内联（`inline`）函数，负责具体的“拷贝”操作。
            *   根据 `docId` 和旧的 `offset`，从旧 `Slice` 中读取完整的属性数据（包括长度信息）。
            *   调用 `Append` 方法将这份数据写入到 `SliceArray` 的末尾（通常是一个新的 `Slice`）。`Append` 会返回一个新的偏移量 `newSliceOffset`。
            *   将新的偏移量编码后，通过 `mOffsetReader->SetOffset(docId, newOffset)` 更新该 `docId` 的偏移量，使其指向新的数据位置。
    *   **设计思想:** `VarNumAttributeDefragSliceArray` 的设计体现了“关注点分离”原则。它本身只负责“整理”这一动作，而将“如何读写 Offset”、“如何解析数据”等具体操作委托给传入的 `Reader` 和 `Formatter` 对象。这种设计使得碎片整理的逻辑可以独立于具体的属性数据格式，具有很好的通用性。
    *   **关键实现:**

        ```cpp
        // in var_num_attribute_defrag_slice_array.h
        inline void VarNumAttributeDefragSliceArray::MoveData(docid_t docId, uint64_t offset)
        {
            // 1. 解码旧的偏移量，找到在 SliceArray 中的物理地址
            uint64_t sliceOffset = mOffsetFormatter.DecodeToSliceArrayOffset(offset);
            const void* data = Get(sliceOffset);
            
            // 2. 从数据头部解析出数据的实际长度
            bool isNull = false;
            size_t size = mDataFormatter.GetDataLength((uint8_t*)data, isNull);
            
            // 3. 将数据追加到 SliceArray 的末尾，获取新的物理地址
            uint64_t newSliceOffset = Append(data, size);
            
            // 4. 将新的物理地址编码成新的偏移量
            uint64_t newOffset = mOffsetFormatter.EncodeSliceArrayOffset(newSliceOffset);
            
            // 5. 更新 docId 对应的偏移量，让它指向新的数据位置
            mOffsetReader->SetOffset(docId, newOffset);
        }

        // in var_num_attribute_defrag_slice_array.cpp
        void VarNumAttributeDefragSliceArray::Defrag(int64_t sliceIdx)
        {
            // ...
            uint64_t offsetBegin = mOffsetFormatter.EncodeSliceArrayOffset(SliceIdxToOffset(sliceIdx));
            uint64_t offsetEnd = offsetBegin + sliceLen;
            docid_t docCount = mOffsetReader->GetDocCount();

            // 遍历所有文档，检查其数据是否在当前要整理的 slice 中
            for (docid_t docId = 0; docId < docCount; docId++) {
                uint64_t offset = mOffsetReader->GetOffset(docId);
                if (offset >= offsetBegin && offset < offsetEnd) {
                    // 如果在，就移动数据
                    MoveData(docId, offset);
                }
            }
            // ...
            // 移动完成后，将旧的 slice 标记为完全浪费
            SetWastedSize(sliceIdx, sliceLen);
            mUselessSliceIdxs.push_back(sliceIdx);
            // ...
        }
        ```

## 3. 技术栈与设计动机

*   **继承与多态:** `VarNumAttributeDefragSliceArray` 继承自 `DefragSliceArray` 并重写了其虚函数（`DoFree`, `Defrag`, `NeedDefrag`），这是典型的面向对象设计，允许上层代码以统一的方式处理不同类型的 `SliceArray` 碎片整理任务。
*   **性能导向:** `MoveData` 函数被声明为 `__ALWAYS_INLINE`，强制内联，以减少函数调用开销。这在 `Defrag` 的循环中被频繁调用，因此性能至关重要。
*   **原子性与一致性:** 整个碎片整理过程需要特别注意数据一致性。`SetOffset` 操作必须是原子的，以确保在任何时刻 `docId` 的偏移量都是有效的，要么指向旧数据，要么指向新数据，不能出现中间状态。这通常由 `AttributeOffsetReader` 的实现来保证（例如，通过 CAS 操作）。
*   **监控与可观测性:** 通过 `AttributeMetrics` 记录了详细的监控指标，如回收的字节数、移动的数据量、浪费的空间大小等。这对于理解系统行为、诊断问题和评估碎片整理的效果至关重要。

## 4. 可能的技术风险与改进方向

*   **Defrag 过程的 IO 和 CPU 开销:** `Defrag` 是一个资源密集型操作。它需要遍历所有文档的 Offset，并进行大量的数据读写。在系统负载很高时执行碎片整理，可能会对在线服务性能造成影响。因此，需要有合理的策略来控制碎片整理的触发时机和执行速度（例如，在低峰期执行，或者限制其 CPU/IO 使用率）。
*   **全量 DocID 扫描:** 当前的 `Defrag` 实现需要遍历全量的 `docId`（从 0 到 `docCount`）。当文档总数非常大时，即使只有一个 `Slice` 需要整理，这个扫描的开销也会非常大。一个潜在的优化是，建立一个“反向索引”（Inverted Index），即从 `sliceIdx` 映射到所有数据在该 `Slice` 内的 `docId` 列表。这样，在整理一个 `Slice` 时，只需要遍历这个列表中的 `docId` 即可，避免了全量扫描。
*   **锁竞争:** `SetOffset` 和 `Append` 操作在多线程环境下可能需要加锁，这会成为性能瓶瓶颈。需要仔细设计锁的粒度，或者采用无锁（Lock-Free）数据结构来优化。

## 5. 总结

`VarNumAttributeDefragSliceArray` 是 Indexlib 存储引擎中一个至关重要的组件，它通过实现一个高效的在线碎片整理机制，解决了变长属性存储中普遍存在的空间浪费问题。其“拷贝-回收”的核心算法虽然经典，但结合具体的业务场景，在实现上做了诸多考量，例如通过委托 `Formatter` 实现通用性，通过 `Metrics` 实现可观测性，以及对性能的极致追求。该模块的设计和实现，充分体现了大型搜索引擎后端在存储系统设计上的深度和精细度。
