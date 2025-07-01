# Indexlib 索引任务框架：扩展与定制机制剖析

## 涉及文件

- `framework/index_task/CustomIndexTaskFactory.h`
- `framework/index_task/CustomIndexTaskFactoryCreator.h`
- `framework/index_task/CustomIndexTaskFactoryCreator.cpp`

## 1. 引言

一个优秀的框架不仅要功能强大，更要易于扩展。Indexlib 的索引任务框架通过一套精巧的工厂和注册机制，实现了高度的可定制性。这使得开发者可以在不修改框架核心代码的情况下，注入自定义的索引任务逻辑，例如实现一种全新的合并策略或一种特殊的索引优化任务。本文将深入剖析这一扩展机制的架构设计、核心组件和工作流程，并提供一个实践指南，帮助开发者理解如何利用该机制扩展 Indexlib 的功能。

## 2. 架构设计：双重工厂与单例注册中心

该扩展机制的核心设计模式是**工厂方法 (Factory Method)** 与 **单例模式 (Singleton)** 的组合，并在此基础上构建了一个“工厂的工厂”架构。

1.  **单例注册中心 (`CustomIndexTaskFactoryCreator`)**: 这是一个全局唯一的单例对象，充当所有自定义任务工厂的“注册中心”。它维护一个从字符串标识符到具体工厂实例的映射。任何模块想要添加一种新的自定义任务类型，都需要先将其对应的工厂注册到这个中心。

2.  **抽象工厂 (`CustomIndexTaskFactory`)**: 这是一个抽象基类，定义了创建一套相关对象（`IIndexTaskPlanCreator` 和 `IIndexOperationCreator`）的接口，但由其子类来决定实例化哪一个具体的类。每一种自定义任务都需要提供一个该接口的具体实现。

3.  **配置驱动**: 整个机制由配置驱动。当系统需要创建一个自定义任务时，它会从任务配置（`IndexTaskConfig`）中读取一个关键的 `class_info` 字段，该字段指定了要使用的工厂类型。系统随后请求单例注册中心提供该类型的工厂，并用它来创建任务所需的计划生成器（Plan Creator）和操作生成器（Operation Creator）。

这个两层工厂的设计，将“知道何时创建”的逻辑（执行引擎）与“知道如何创建”的逻辑（具体工厂实现）完美地分离开来，实现了高度的解耦和灵活性。

## 3. 核心组件与工作流程

### 3.1. `CustomIndexTaskFactoryCreator`：全局唯一的工厂管理器

`CustomIndexTaskFactoryCreator` 是整个扩展机制的入口点。它被实现为一个线程安全的单例，确保在整个进程空间中只有一个实例。

-   **职责**: 维护一个 `std::map<std::string, CustomIndexTaskFactory*>`，存储所有已注册的工厂。它提供了 `Register` 和 `Create` 两个核心方法。
-   **`Register`**: 将一个工厂实例与一个唯一的字符串名称关联起来。如果注册了重复的名称，程序会 `abort`，以防止不明确的行为。
-   **`Create`**: 根据给定的名称查找并返回一个工厂的克隆实例。注意，返回的是 `unique_ptr<CustomIndexTaskFactory>`，调用者拥有其生命周期。

关键代码实现 (`CustomIndexTaskFactoryCreator.cpp`):

```cpp
// CustomIndexTaskFactoryCreator 是一个单例
class CustomIndexTaskFactoryCreator : public indexlib::util::Singleton<CustomIndexTaskFactoryCreator>
{
    // ...
public:
    void Register(std::string factoryName, CustomIndexTaskFactory* factory);
    std::unique_ptr<CustomIndexTaskFactory> Create(std::string factoryName) const;
    // ...
private:
    std::map<std::string, CustomIndexTaskFactory*> _factorysMap;
    mutable autil::ThreadMutex _mutex;
};

void CustomIndexTaskFactoryCreator::Register(std::string tableType, CustomIndexTaskFactory* factory)
{
    autil::ScopedLock lock(_mutex);
    auto iter = _factorysMap.find(tableType);
    if (iter != _factorysMap.end()) {
        std::abort(); // 不允许重复注册
    }
    _factorysMap[tableType] = factory;
}

std::unique_ptr<CustomIndexTaskFactory> CustomIndexTaskFactoryCreator::Create(std::string tableType) const
{
    autil::ScopedLock lock(_mutex);
    auto iter = _factorysMap.find(tableType);
    if (iter == _factorysMap.end()) {
        return nullptr;
    }
    return iter->second->Create(); // 返回工厂的克隆
}
```

### 3.2. `REGISTER_CUSTOM_INDEX_TASK_FACTORY`：自动注册的魔法

为了让开发者能够轻松地注册自己的工厂，框架提供了一个非常便利的宏：`REGISTER_CUSTOM_INDEX_TASK_FACTORY`。

```cpp
#define REGISTER_CUSTOM_INDEX_TASK_FACTORY(FACTORY_TYPE, TASK_FACTORY)                                                 \
    __attribute__((constructor)) void Register##FACTORY_TYPE##Factory()                                                \
    {                                                                                                                  \
        auto customIndexTaskFactoryCreator = indexlibv2::framework::CustomIndexTaskFactoryCreator::GetInstance();      \
        customIndexTaskFactoryCreator->Register(#FACTORY_TYPE, (new TASK_FACTORY));                                    \
    }
```

这个宏是整个机制能够实现“即插即用”的关键。它利用了 GCC/Clang 的一个特性 `__attribute__((constructor))`。被这个属性修饰的函数会在 `main` 函数执行之前，或者在动态链接库（`.so`）被加载时自动执行。这意味着，开发者只需要在他们的自定义工厂的 `.cpp` 文件中调用这个宏，链接器就会确保在程序启动时，这个工厂被自动注册到 `CustomIndexTaskFactoryCreator` 中，无需任何显式的初始化调用。

### 3.3. `CustomIndexTaskFactory`：连接配置与实现

`CustomIndexTaskFactory` 提供了两个核心的静态方法，它们是框架代码与自定义代码之间的桥梁。

```cpp
// in CustomIndexTaskFactory.h

static std::pair<Status, std::unique_ptr<IIndexTaskPlanCreator>>
GetCustomPlanCreator(const std::shared_ptr<config::IndexTaskConfig>& indexTaskConfig);

static std::pair<Status, std::unique_ptr<framework::IIndexOperationCreator>>
GetCustomOperationCreator(const std::shared_ptr<config::IndexTaskConfig>& indexTaskConfig,
                          const std::shared_ptr<config::ITabletSchema>& schema);
```

这两个方法的工作流程如下：

1.  从传入的 `IndexTaskConfig` 中尝试解析 `class_info` 配置项。
2.  如果 `class_info` 存在，就获取其中定义的类名（即工厂名）。
3.  使用这个名称，向 `CustomIndexTaskFactoryCreator` 单例请求对应的工厂实例。
4.  如果成功获取工厂，就调用其 `CreateIndexTaskPlanCreator` 或 `CreateIndexOperationCreator` 方法来创建所需的具体实例。
5.  如果 `class_info` 不存在，则表示这是一个内置任务，返回 `nullptr`，上层逻辑会使用默认的创建器。

## 4. 如何添加一个新的自定义任务：实践指南

假设我们要实现一个名为 `MyOptimize` 的自定义合并任务，步骤如下：

1.  **实现自定义的 Plan 和 Operation Creator**: 
    -   `class MyOptimizeTaskPlanCreator : public IIndexTaskPlanCreator { ... };`
    -   `class MyOptimizeOperationCreator : public IIndexOperationCreator { ... };`

2.  **实现自定义的 Factory**:

    ```cpp
    // MyOptimizeTaskFactory.h
    class MyOptimizeTaskFactory : public CustomIndexTaskFactory {
    public:
        std::unique_ptr<CustomIndexTaskFactory> Create() const override { return std::make_unique<MyOptimizeTaskFactory>(*this); }
        std::unique_ptr<IIndexOperationCreator> CreateIndexOperationCreator(...) override { return std::make_unique<MyOptimizeOperationCreator>(); }
        std::unique_ptr<IIndexTaskPlanCreator> CreateIndexTaskPlanCreator() override { return std::make_unique<MyOptimizeTaskPlanCreator>(); }
    };
    ```

3.  **注册自定义 Factory**:

    ```cpp
    // MyOptimizeTaskFactory.cpp
    #include "MyOptimizeTaskFactory.h"

    REGISTER_CUSTOM_INDEX_TASK_FACTORY(MyOptimize, MyOptimizeTaskFactory);
    ```

4.  **修改编译配置 (BUILD)**: 确保包含 `MyOptimizeTaskFactory.cpp` 的 `cc_library` 目标设置了 `alwayslink = True`。这可以防止链接器认为注册代码是“无用代码”而将其优化掉。

5.  **配置任务**: 在 Tablet 的 JSON 配置中，添加或修改任务配置：

    ```json
    "index_task_configs": [
        {
            "task_name": "my_optimize_task",
            "task_type": "merge",
            "trigger": "manual",
            "settings": {
                "class_info": {
                    "class_name": "MyOptimize"
                }
            }
        }
    ]
    ```

完成以上步骤后，当触发名为 `my_optimize_task` 的任务时，Indexlib 框架就会自动加载并使用你定义的 `MyOptimizeTaskPlanCreator` 和 `MyOptimizeOperationCreator`。

## 5. 技术风险与考量

-   **命名冲突**: 工厂注册依赖于字符串名称。如果不同的模块或团队使用了相同的工厂名称，会导致程序在启动时 `abort`。需要有统一的命名规范来避免冲突。
-   **链接器依赖**: `alwayslink = True` 的设置至关重要，但容易被遗忘，导致难以排查的“工厂未注册”错误。这增加了对开发者构建系统知识的要求。
-   **单例的生命周期**: 虽然 `CustomIndexTaskFactoryCreator` 是一个单例，但它所持有的 `CustomIndexTaskFactory` 指针是在程序启动时 `new` 出来的，并在 `Creator` 的析构函数中 `delete`。这在正常的程序生命周期中没有问题，但在复杂的测试环境或动态卸载库的场景下，需要注意内存管理。

## 6. 总结

Indexlib 的扩展与定制机制是其框架设计的一大亮点。通过巧妙地结合单例、工厂模式和编译器特性，它提供了一个强大、灵活且对开发者友好的插件系统。开发者可以聚焦于任务本身的业务逻辑，而无需关心底层的调度和执行细节，极大地提高了开发效率和系统的可维护性。
