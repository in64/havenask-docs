
# Indexlib Table 并发与资源管理

**涉及文件:**
* `indexlib/table/executor_provider.h`
* `indexlib/table/executor_provider.cpp`
* `indexlib/table/executor_manager.h`
* `indexlib/table/executor_manager.cpp`

## 1. 系统概述

在现代搜索引擎和数据处理系统中，为了充分利用多核 CPU 的计算能力，**并发**编程是必不可少的。Indexlib 在其 `table` 模块中，也引入了一套并发与资源管理机制，用于支持索引的并行构建、查询和合并。这套机制的核心是 `future_lite::Executor`，一个用于执行异步任务的执行器（通常是线程池）。

本篇文档将深入分析 `table` 模块中与并发和资源管理相关的代码，重点阐述 `ExecutorProvider` 和 `ExecutorManager` 的设计与实现，揭示 Indexlib 如何管理和复用 `Executor` 这一宝贵的系统资源。

## 2. `future_lite::Executor`：并发执行的基石

在分析 `ExecutorProvider` 和 `ExecutorManager` 之前，我们首先需要了解 `future_lite::Executor`。`future_lite` 是一个轻量级的、用于异步编程的库，其核心组件 `Executor` 扮演着**任务调度器和执行器**的角色。你可以将一个 `Executor` 简单地理解为一个**线程池**。

通过将任务提交给 `Executor`，我们可以实现任务的并行执行，从而提高系统的吞吐量和响应速度。在 Indexlib 中，`Executor` 被广泛应用于各种需要并行处理的场景，例如：

*   **并行查询**: `TableReader` 可以利用 `Executor`，将一个复杂的查询分解为多个子查询，并并行地在不同的 Segment 上执行。
*   **并行合并**: `TableMerger` 可以利用 `Executor`，并行地处理不同的数据分片或执行不同的合并任务。

## 3. `ExecutorProvider`：`Executor` 的提供者

`ExecutorProvider` 是一个抽象基类，它的职责是**创建和提供**特定类型的 `Executor` 实例。`ExecutorProvider` 的设计体现了**依赖注入**的思想：它将 `Executor` 的创建逻辑，从使用 `Executor` 的组件（如 `TableReader`）中解耦出来。

### 关键设计与实现

*   **抽象接口**: `ExecutorProvider` 定义了两个纯虚函数，要求其子类必须实现：
    *   `GetExecutorName()`: 返回该 `ExecutorProvider` 提供的 `Executor` 的名称。这个名称将作为 `Executor` 的唯一标识。
    *   `CreateExecutor()`: 创建并返回一个 `future_lite::Executor` 的实例。

*   **初始化 (`Init`)**: `ExecutorProvider` 的 `Init` 方法接收 `Schema` 和 `Options` 作为参数。这使得子类在创建 `Executor` 时，可以根据索引的配置信息，来决定 `Executor` 的类型和参数（例如，线程池的大小、队列长度等）。

通过继承 `ExecutorProvider`，不同的 Table 类型或不同的业务场景，可以实现自定义的 `Executor` 创建逻辑。例如，对于计算密集型的任务，可以创建一个固定大小的线程池；而对于 I/O 密集型的任务，则可以创建一个可动态调整大小的线程池。

#### 核心代码示例 (`executor_provider.h`)

```cpp
class ExecutorProvider
{
public:
    ExecutorProvider();
    virtual ~ExecutorProvider();

public:
    bool Init(const config::IndexPartitionSchemaPtr& schema, const config::IndexPartitionOptions& options);
    virtual bool DoInit() = 0;
    virtual std::string GetExecutorName() const = 0;
    virtual future_lite::Executor* CreateExecutor() const = 0;

protected:
    config::IndexPartitionSchemaPtr mSchema;
    config::IndexPartitionOptions mOptions;
};
```

## 4. `ExecutorManager`：`Executor` 的管理者与复用池

如果系统中的每个组件都独立地创建和管理自己的 `Executor`，将会导致线程资源的严重浪费和竞争。为了解决这个问题，Indexlib 引入了 `ExecutorManager`，一个用于**统一管理和复用** `Executor` 实例的组件。

`ExecutorManager` 的核心思想是构建一个 `Executor` 的**共享池**。它根据 `Executor` 的名称（由 `ExecutorProvider::GetExecutorName()` 提供）来缓存和复用 `Executor` 实例。

### 关键设计与实现

*   **注册与获取 (`RegisterExecutor`)**: `ExecutorManager` 的 `RegisterExecutor` 方法接收一个 `ExecutorProvider` 作为参数。其工作流程如下：
    1.  首先，从 `ExecutorProvider` 获取 `Executor` 的名称。
    2.  然后，在内部的 `mExecutors`（一个 `map`）中，查找是否已存在同名的 `Executor`。
    3.  如果存在，则直接返回缓存的 `Executor` 实例（的 `shared_ptr`）。
    4.  如果不存在，则调用 `ExecutorProvider` 的 `CreateExecutor` 方法创建一个新的 `Executor` 实例，将其存入 `mExecutors` 中，并返回该实例。

*   **线程安全**: `RegisterExecutor` 方法使用了 `autil::ThreadMutex` 来保证对 `mExecutors` 的访问是线程安全的。这使得多个线程可以同时向 `ExecutorManager` 注册和获取 `Executor`，而不会发生数据竞争。

*   **生命周期管理与资源回收 (`ClearUselessExecutors`)**: `ExecutorManager` 使用 `std::shared_ptr` 来管理 `Executor` 的生命周期。当一个 `Executor` 不再被任何组件使用时（即其 `shared_ptr` 的引用计数变为 1，只剩下 `ExecutorManager` 自身持有的那一个引用），`ClearUselessExecutors` 方法会将其从 `mExecutors` 中移除并销毁，从而回收线程池等相关资源。

通过 `ExecutorManager`，Indexlib 实现 `Executor` 资源的**按需创建**和**全局共享**，有效地避免了线程资源的浪费，提高了系统的整体性能和可伸缩性。

#### 核心代码示例 (`executor_manager.h`)

```cpp
class ExecutorManager
{
public:
    ExecutorManager();
    ~ExecutorManager();

public:
    std::shared_ptr<future_lite::Executor> RegisterExecutor(const std::shared_ptr<ExecutorProvider>& provider);
    size_t ClearUselessExecutors();

private:
    mutable autil::ThreadMutex mMapLock;
    std::map<std::string, std::shared_ptr<future_lite::Executor>> mExecutors;
};
```

## 5. 技术考量与设计权衡

*   **`Executor` 的粒度**: `ExecutorManager` 的设计是基于 `Executor` 名称的。因此，如何定义 `Executor` 的名称和粒度，是一个需要权衡的问题。如果粒度过细（例如，每个 `TableReader` 实例都使用一个不同名称的 `Executor`），会导致 `Executor` 无法被有效复用。如果粒度过粗（例如，所有组件都使用同一个 `Executor`），则可能会导致不同类型的任务之间相互干扰。合理的做法是，根据任务的类型和特性，定义有限的几种 `Executor`（例如，`io_executor`、`cpu_executor`），并让不同的组件根据需要选择使用。

*   **资源隔离**: 虽然 `ExecutorManager` 实现了 `Executor` 的共享，但在某些场景下，可能需要进行资源隔离，以防止某些高负载的任务耗尽所有线程资源，影响其他任务的执行。在这种情况下，可以通过实现自定义的 `ExecutorProvider`，并为其指定一个唯一的名称，来创建一个独占的 `Executor`。

*   **与 `future_lite` 的集成**: `ExecutorProvider` 和 `ExecutorManager` 的设计，与 `future_lite` 库紧密耦合。这使得 Indexlib 可以充分利用 `future_lite` 提供的异步编程能力。但同时，也意味着对 `future_lite` 的依赖较强。

## 6. 总结

Indexlib 的**并发与资源管理**机制，通过 `ExecutorProvider` 和 `ExecutorManager` 这两个核心组件，构建了一套高效、可扩展的 `Executor` 管理框架。`ExecutorProvider` 将 `Executor` 的创建逻辑解耦出来，提供了很好的灵活性。而 `ExecutorManager` 则通过共享池的方式，实现了 `Executor` 资源的统一管理和高效复用。这套机制为 Indexlib 的并行化处理能力提供了坚实的基础，是其高性能、高吞吐量特性的重要保障。
