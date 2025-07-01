
# Indexlib 策略组合、辅助与拆分（Combined, Utility & Split Strategies）深度解析

**涉及文件:**
*   `table/normal_table/index_task/merger/CombinedMergeStrategy.cpp`
*   `table/normal_table/index_task/merger/CombinedMergeStrategy.h`
*   `table/normal_table/index_task/merger/SplitStrategy.cpp`
*   `table/normal_table/index_task/merger/SplitStrategy.h`
*   `table/normal_table/index_task/merger/NormalTableMergeStrategyUtil.cpp`
*   `table/normal_table/index_task/merger/NormalTableMergeStrategyUtil.h`
*   `table/normal_table/index_task/merger/NormalTabletMergeStrategyDefine.cpp`
*   `table/normal_table/index_task/merger/NormalTabletMergeStrategyDefine.h`

## 1. 概述：合并策略的“生态系统”

如果说核心合并策略算法（如 `BalanceTree`, `Adaptive` 等）是 Indexlib 合并决策的“引擎”，那么本文档所解析的这一系列组件，就是支撑这些引擎高效、灵活运转的“传动系统”、“工具箱”和“逆变器”。它们共同构成了 Indexlib 合并策略的完整“生态系统”，使得单一、独立的策略能够被有机地组合、复用和扩展，以应对更加复杂和多样化的现实场景。

这个生态系统主要由三类角色构成：

1.  **指挥官 (`CombinedMergeStrategy`)**: 负责编排和调度多个独立的合并策略。它允许用户定义一个“策略链”，让不同的策略在不同的时机、针对不同类型的 `Segment` 发挥作用，实现“1+1>2”的效果。
2.  **工兵 (`SplitStrategy`)**: 在特定场景下，执行与合并完全相反的操作——`Segment` 拆分。它如同特种工兵，负责处理将一个过大的 `Segment` 分解成多个小 `Segment` 的特殊任务，以满足负载均衡或数据分发的需求。
3.  **后勤 (`NormalTableMergeStrategyUtil`, `NormalTabletMergeStrategyDefine`)**: 提供所有策略都会用到的公共函数、数据结构和常量定义。它们是代码复用、逻辑统一和可维护性的基石，为整个合并策略子系统提供着不可或缺的后勤支援。

通过这个生态系统，Indexlib 的合并框架不再仅仅是几个孤立算法的集合，而是一个具备高度灵活性、可扩展性和完备性的精密体系。

## 2. 核心组件详解

### 2.1. 组合策略 (`CombinedMergeStrategy`)：策略的“元控制器”

在真实的生产环境中，单一的合并策略往往难以完美应对所有情况。例如，我们可能希望在白天业务高峰期采用保守的增量合并策略（如 `PriorityQueue`），以降低对在线服务的影响；而在午夜系统空闲时，则执行一次彻底的优化合并（`Optimize`）。`CombinedMergeStrategy` 正是为实现这种复杂编排而设计的。

**设计思想**: 作为一个“元策略”（Meta-Strategy）或“策略的容器”，它本身不包含任何具体的 `Segment` 选择逻辑。它的唯一职责是按照预定义的顺序，依次调用其内部包含的一系列子策略（`Sub-Strategy`），并将它们生成的 `MergePlan` 汇总起来，形成最终的合并决策。

**核心逻辑与算法**:
1.  **加载子策略**: `CombinedMergeStrategy` 在初始化时，会根据 `JSON` 配置，创建并持有一组子策略的实例。配置中会明确定义每个子策略的类型及其具体参数。
2.  **顺序调用**: 当 `CreateMergePlan` 方法被调用时，它会遍历其内部的子策略列表。
3.  **上下文传递与隔离**: 对于每个子策略，`CombinedMergeStrategy` 会调用其 `CreateMergePlan` 方法。关键在于，它需要处理好 `Segment` 的上下文传递。当一个子策略生成了一个 `MergePlan` 后，这个计划中涉及的 `Segment` 就不应该再被后续的子策略考虑了。`CombinedMergeStrategy` 会从总的 `Segment` 列表中“减去”已被前序策略选中合并的 `Segment`，然后将剩余的 `Segment` 传递给下一个子策略。
4.  **汇总计划**: 它会将所有子策略返回的有效 `MergePlan` 收集起来，合并成一个大的、最终的 `MergePlan` 返回给上层调用者。

**关键实现细节**:

```cpp
MergePlan CombinedMergeStrategy::CreateMergePlan(const framework::SegmentDescriptions& segDescs)
{
    MergePlan combinedPlan;
    framework::SegmentDescriptions remainingSegDescs = segDescs;

    // 1. 遍历所有子策略
    for (const auto& strategy : _subStrategies) {
        if (remainingSegDescs.empty()) {
            break;
        }

        // 2. 为子策略创建计划
        MergePlan subPlan = strategy->CreateMergePlan(remainingSegDescs);

        if (!subPlan.empty()) {
            // 3. 将子计划加入总计划
            for (const auto& task : subPlan.GetMergeTasks()) {
                combinedPlan.AddTask(task);
            }

            // 4. 更新剩余的 Segment 列表，供下一个策略使用
            std::set<segmentid_t> mergedSegments = subPlan.GetAllSegmentIds();
            framework::SegmentDescriptions nextRemainingSegDescs;
            for (const auto& segDesc : remainingSegDescs) {
                if (mergedSegments.find(segDesc.GetSegmentId()) == mergedSegments.end()) {
                    nextRemainingSegDescs.push_back(segDesc);
                }
            }
            remainingSegDescs = nextRemainingSegDescs;
        }
    }
    return combinedPlan;
}
```

**设计价值与风险**:
*   **价值**: 提供了无与伦比的灵活性。通过自由组合不同的基础策略，用户可以定制出高度贴合自身业务模式的复杂合并逻辑，而无需编写任何新的代码。
*   **风险**: 配置复杂性增加。不合理的策略组合或顺序，可能会导致意想不到的行为，例如某个策略永远得不到执行，或者产生冲突的合并计划。需要用户对各个子策略的特性有深入的理解。

### 2.2. 拆分策略 (`SplitStrategy`)：合并的逆向工程

在绝大多数场景下，我们都需要将小 `Segment` 合并成大 `Segment`。但在某些特殊情况下，反向操作——将一个大 `Segment` 拆分成多个小 `Segment`——也同样重要。

**功能目标**: 接收一个大的源 `Segment`，根据指定的拆分规则（如按文档数、按主键范围），生成一个“拆分计划”（在 Indexlib 中，这同样由 `MergePlan` 结构来表达，只是其含义不同），指导后续操作将这个 `Segment` 的数据重新组织到多个新的、更小的 `Segment` 中。

**核心逻辑与算法**:
1.  **获取拆分目标**: 策略需要知道要拆分哪个 `Segment`，以及要拆分成多少份（`split_count`）。这通常由外部任务（如运维指令）传入。
2.  **计算拆分点**: 策略的核心是计算出文档级别的“拆分点”。例如，如果要将一个包含 10000 个文档的 `Segment` 拆分成 5 份，那么拆分点就是 `docid=2000, 4000, 6000, 8000`。
3.  **生成“伪合并计划”**: `SplitStrategy` 会巧妙地复用 `MergePlan` 的数据结构来描述拆分任务。它会创建一个 `MergePlan`，其中：
    *   源 `Segment` 只有一个，即待拆分的 `Segment`。
    *   目标 `Segment` 有多个（等于 `split_count`）。
    *   最关键的是，它会为每个目标 `Segment` 指定其对应的文档范围（`docid_range`）。
4.  **后续操作的配合**: 后续的合并执行 `Operation`（如 `AttributeIndexMergeOperation`）在看到这样一个特殊的 `MergePlan` 后，会理解其为拆分任务。它们在执行数据拷贝时，会根据 `docid_range`，只将源 `Segment` 的一部分文档拷贝到对应的目标 `Segment` 中，从而完成物理上的拆分。

**关键实现细节**:

```cpp
// SplitStrategy::CreateMergePlan 伪代码逻辑
MergePlan SplitStrategy::CreateMergePlan(const framework::SegmentDescriptions& segDescs)
{
    // 1. 从参数中获取要拆分的 segmentId 和拆分数 splitCount
    segmentid_t sourceSegmentId = _params.Get("source_segment_id");
    int splitCount = _params.Get("split_count");

    // 2. 找到源 Segment 的描述信息
    auto sourceSegmentDesc = FindSegment(segDescs, sourceSegmentId);
    if (!sourceSegmentDesc) {
        return MergePlan();
    }
    uint32_t docCount = sourceSegmentDesc->GetDocCount();

    // 3. 计算每个新 Segment 的文档数
    uint32_t docsPerSegment = docCount / splitCount;

    MergePlan plan;
    MergeTask task;
    task.AddSourceSegment(sourceSegmentId);

    // 4. 为每个新 Segment 定义文档范围
    docid_t currentDocId = 0;
    for (int i = 0; i < splitCount; ++i) {
        docid_t endDocId = (i == splitCount - 1) ? docCount : currentDocId + docsPerSegment;
        task.AddTargetSegment().SetDocIdRange({currentDocId, endDocId});
        currentDocId = endDocId;
    }
    plan.AddTask(task);
    return plan;
}
```

**适用场景**:
*   **数据迁移与负载均衡**: 当需要将一个大索引分发到多个机器上时，可以先将其拆分成多个小的 `Segment`，再进行分发。
*   **并行查询**: 在某些MPP（大规模并行处理）查询引擎中，将大 `Segment` 拆分后，可以并行地在多个计算节点上查询，提升大查询的吞吐量。

### 2.3. 公共辅助与定义 (`NormalTableMergeStrategyUtil`, `NormalTabletMergeStrategyDefine`)

这两个组件是典型的“幕后英雄”，它们不直接参与决策，但为所有策略的实现提供了便利和规范。

*   **`NormalTabletMergeStrategyDefine`**: 这个模块主要负责**定义**。
    *   **常量定义**: 定义了所有合并策略 `JSON` 配置中的 key 名称，如 `"max-merged-segment-count"`, `"merge-factor"` 等。这避免了在代码中硬编码字符串，提高了代码的可读性和可维护性。当需要修改配置项名称时，只需修改一处定义即可。
    *   **结构体定义**: 定义了策略之间传递参数或返回结果时可能用到的数据结构。

*   **`NormalTableMergeStrategyUtil`**: 这个模块是一个**静态工具类**的集合。
    *   **公共算法封装**: 封装了被多个策略重复使用的算法逻辑。例如，`SelectSegmentsByDocCount` 函数，可以根据 `Segment` 的文档数，从一个列表中筛选出符合条件的 `Segment`。`PriorityQueueMergeStrategy` 和 `BalanceTreeMergeStrategy` 都可能用到类似的逻辑。
    *   **配置解析**: 提供了从 `JSON` 参数中安全地提取各种类型值的工具函数，并能处理默认值和异常情况。
    *   **校验与日志**: 提供了用于校验合并计划合法性、打印调试日志等辅助功能。

**设计价值**:
*   **高内聚、低耦合**: 将公共逻辑下沉到 `Util` 类，避免了不同策略模块之间的直接依赖和代码重复，使得每个策略的实现更加聚焦于其自身的核心算法。
*   **代码健壮性与可维护性**: 通过统一的定义和工具，保证了配置解析、参数传递的一致性和正确性，减少了因编码不规范或考虑不周而引入的 Bug。

## 3. 结论

`CombinedMergeStrategy`、`SplitStrategy` 以及一系列的 `Util` 和 `Define` 组件，共同构成了 Indexlib 合并策略框架的“软实力”。它们将一个个独立的算法“粘合”成一个强大而灵活的整体，展现了优秀软件框架的设计之道：

*   **组合优于继承**: 通过 `CombinedMergeStrategy`，实现了对策略行为的灵活组合，远比通过复杂的继承关系来扩展功能要更加优雅和强大。
*   **正交性设计**: 合并与拆分作为两个正交的功能，被分别实现，但又可以统一在 `MergePlan` 的框架下进行描述，体现了设计的统一性。
*   **Don't Repeat Yourself (DRY)**: `Util` 和 `Define` 模块是 DRY 原则的最佳实践，它们最大限度地消除了代码冗余，提升了整个系统的开发效率和质量。

对这个“生态系统”的理解，是超越“了解某个具体合并算法”的更高层次的认知。它揭示了 Indexlib 如何从框架层面，为用户提供一个既强大又易于扩展的合并决策系统。
