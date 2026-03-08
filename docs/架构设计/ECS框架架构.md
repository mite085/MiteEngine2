# ECS 框架架构

**项目**：MiteEngine2 - SDF Raymarching 3D渲染引擎
**版本**：v1.0
**角色**：模块架构师（ECS框架）
**日期**：2026年3月9日

---

## 1. 模块概述

### 1.1 ECS模块定位

ECS（Entity-Component-System）是MiteEngine2的核心架构模式，负责组织和管理游戏运行时实体。ECS模块作为连接Core与其他业务模块（Rendering、Physics、Resources）的枢纽，提供数据驱动的游戏逻辑组织方式。

### 1.2 技术选型

| 项目 | 选择 | 理由 |
|------|------|------|
| ECS库 | EnTT 3.12+ | Header-only、高性能、缓存友好、社区活跃 |
| 实体类型 | EnTT entity | 轻量句柄，32位ID |
| 组件存储 | EnTT registry | 紧凑存储，自动内存管理 |
| 系统调度 | 自定义调度器 | 支持依赖排序和多线程 |

### 1.3 命名空间约定

```cpp
namespace mte {
namespace ecs {

namespace entity { }
namespace component { }
namespace system { }
namespace event { }

} // namespace ecs
} // namespace mte
```

---

## 2. 实体管理

### 2.1 实体概念

实体是轻量级句柄，不包含任何数据，仅作为组件的容器和标识。

```cpp
namespace mte::ecs {

using Entity = entt::entity;
using EntityHandle = entt::entity;

class EntityManager {
public:
    Entity CreateEntity();
    Entity CreateEntity(const std::string& name);
    void DestroyEntity(Entity entity);
    bool IsValid(Entity entity) const;
    size_t GetEntityCount() const;

private:
    entt::registry registry_;
    std::unordered_map<Entity, std::string> entityNames_;
};
```

### 2.2 实体查询API

```cpp
class EntityQuery {
public:
    template<typename... Components>
    auto GetView();

    template<typename... Components>
    size_t GetCount();

    template<typename... Components, typename Func>
    void ForEach(Func func);
};
```

---

## 3. 组件系统

### 3.1 组件类型定义

组件是纯数据容器，不包含业务逻辑。

```cpp
// 核心组件示例
struct Transform {
    Vec3 position{0, 0, 0};
    Vec3 rotation{0, 0, 0};
    Vec3 scale{1, 1, 1};
};

struct Name {
    std::string name;
};

// 相位感知组件
struct PhaseComponent {
    uint8_t currentPhase;
    uint8_t availablePhases;
    uint8_t visiblePhases;
    uint8_t collisionPhases;
};
```

### 3.2 组件存储策略

| 策略 | 适用场景 |
|------|---------|
| BasicComponent | 大多数组件，高性能 |
| DenseComponent | 大数据量组件 |
| SparseComponent | 组件存在率低 |
| ExternalComponent | 第三方内存管理 |

---

## 4. 系统调度

### 4.1 系统基类定义

```cpp
class ISystem {
public:
    virtual ~ISystem() = default;
    virtual void Initialize(World& world);
    virtual void Update(float deltaTime);
    virtual void PreRender(float deltaTime);
    virtual void PostRender(float deltaTime);
    virtual void Shutdown();
    virtual int GetPriority() const;
    virtual std::string GetName() const;
    virtual bool IsEnabled() const;
    virtual void SetEnabled(bool enabled);

protected:
    World* world_ = nullptr;
    bool enabled_ = true;
};
```

### 4.2 执行顺序规划

| 优先级 | 系统 | 职责 |
|--------|------|------|
| 50 | InputSystem | 玩家输入处理 |
| 100 | PhaseSystem | 相位切换逻辑 |
| 200 | AnimationSystem | 动画更新 |
| 300 | MovementSystem | 移动计算 |
| 400 | PhysicsSystem | 物理模拟 |
| 500 | CollisionSystem | 碰撞检测 |
| 800 | TransformSystem | 变换同步 |
| 900 | RenderSystem | 渲染准备 |
| 1000 | PresentSystem | 提交渲染 |

---

## 5. 组件分类

### 5.1 核心组件

| 组件 | 描述 |
|------|------|
| EntityId | 实体唯一标识 |
| Name | 实体名称 |
| Transform | 位置/旋转/缩放 |
| Tags | 标签集合 |
| Layer | 层级 |
| Enabled | 启用状态 |

### 5.2 渲染组件

| 组件 | 描述 |
|------|------|
| SDFGeometry | SDF根节点指针 |
| Material | 材质引用 |
| Light | 光源属性 |
| Camera | 摄像机属性 |
| Renderable | 渲染标记 |

### 5.3 物理组件

| 组件 | 描述 |
|------|------|
| Collider | 碰撞体定义 |
| RigidBody | 刚体属性 |
| Trigger | 触发器标记 |

### 5.4 相位组件

| 组件 | 描述 |
|------|------|
| PhaseComponent | 相位信息 |
| PhaseInteraction | 相位交互规则 |
| PhaseTransition | 相位过渡动画 |

---

## 6. 事件系统

### 6.1 事件类型

```cpp
struct EntityCreated { Entity entity; };
struct EntityDestroyed { Entity entity; };
struct PhaseChanged { Entity entity; uint8_t oldPhase; uint8_t newPhase; };
struct CollisionOccurred { Entity entityA; Entity entityB; Vec3 contactPoint; };
```

### 6.2 事件分发器

```cpp
class EventDispatcher {
public:
    template<typename EventType>
    void Subscribe(std::function<void(const EventType&)> handler);

    template<typename EventType>
    void Dispatch(const EventType& event);

    template<typename EventType>
    void Enqueue(const EventType& event);

    void ProcessQueue();
};
```

---

## 7. World 管理器

```cpp
class World {
public:
    Entity CreateEntity();
    void DestroyEntity(Entity entity);

    template<typename T>
    T& AddComponent(Entity e, T&& component);

    template<typename T>
    T* GetComponent(Entity e);

    template<typename... T>
    auto View();

    void AddSystem(std::unique_ptr<ISystem> system);
    void Update(float deltaTime);

private:
    entt::registry registry_;
    SystemScheduler scheduler_;
    EventDispatcher eventDispatcher_;
};
```

---

## 8. 文件组织

```
ECS/
├── Entity/
│   ├── Entity.hpp
│   └── EntityManager.hpp
├── Component/
│   ├── Core/ (Transform, Name, Tags)
│   ├── Rendering/ (SDFGeometry, Material, Light)
│   ├── Physics/ (Collider, RigidBody)
│   └── Gameplay/ (PhaseComponent)
├── System/
│   ├── ISystem.hpp
│   └── SystemScheduler.hpp
├── Event/
│   ├── Event.hpp
│   └── EventDispatcher.hpp
└── World/
    └── World.hpp
```

---

**文档结束**
