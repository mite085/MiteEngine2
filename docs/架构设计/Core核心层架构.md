# Core 核心层架构

**项目**：MiteEngine2 - SDF Raymarching 3D渲染引擎
**版本**：v1.0
**角色**：模块架构师（Core核心层）
**日期**：2026年3月9日

---

## 1. 模块概述

### 1.1 Core模块定位

Core 是引擎的最底层基础模块，**无任何外部依赖**（除C++标准库外）。所有上层模块（Platform、ECS、Rendering、Physics、Resources、Editor）都依赖Core模块。

### 1.2 设计原则

| 原则 | 描述 |
|------|------|
| **零外部依赖** | 只使用C++标准库，不依赖第三方库 |
| **性能优先** | 核心路径零抽象开销，支持内联优化 |
| **稳定优先** | 公共API一旦发布，不轻易变更 |
| **可测试性** | 核心功能必须可单元测试 |

### 1.3 子系统划分

```
Core/
├── Memory/        # 内存管理（分配器、智能指针、调试）
├── Math/          # 数学库（向量、矩阵、四元数、几何）
├── Thread/        # 线程与并发（线程池、原子操作、同步原语）
├── FileSystem/    # 文件系统（路径、读写、虚拟文件系统）
├── Logger/        # 日志系统（级别、输出、格式化）
└── Container/     # 容器工具（可选扩展）
```

### 1.4 命名空间约定

所有Core模块代码在 `mte` 命名空间下：

```cpp
namespace mte {

// 子模块使用子命名空间
namespace memory { }
namespace math { }
namespace thread { }
namespace filesystem { }
namespace logger { }

} // namespace mte
```

---

## 2. Memory 子系统

### 2.1 设计目标

- 提供多种内存分配策略（线性、池式、堆式）
- 零开销抽象的基础类型
- 内存追踪和调试支持
- 与上层模块无缝集成

### 2.2 内存分配器

#### 2.2.1 分配器接口

```cpp
namespace mte::memory {

class IAllocator {
public:
    virtual ~IAllocator() = default;

    virtual void* Allocate(size_t size, size_t alignment = 0) = 0;
    virtual void Deallocate(void* ptr) = 0;
    virtual size_t GetAllocatedSize() const = 0;
    virtual size_t GetTotalAllocated() const = 0;
};

// 智能指针工厂
class AllocatorFactory {
public:
    static UniquePtr<T> CreateUnique(UniquePtr<T>*, IAllocator* alloc);
    static SharedPtr<T> CreateShared(SharedPtr<T>*, IAllocator* alloc);
};

} // namespace mte::memory
```

#### 2.2.2 线性分配器（LinearAllocator）

**用途**：帧分配、临时数据、解析器

```cpp
class LinearAllocator : public IAllocator {
public:
    explicit LinearAllocator(size_t bufferSize);
    ~LinearAllocator() override;

    void* Allocate(size_t size, size_t alignment = alignof(max_align_t)) override;
    void Deallocate(void* ptr) override;

    // 重置所有分配（O(1)）
    void Reset();

    // 获取当前指针位置
    void* GetCurrentPtr() const;
    void SetCurrentPtr(void* ptr);

private:
    char* buffer_;
    char* current_;
    size_t bufferSize_;
};
```

**使用示例**：

```cpp
// 每帧创建临时数据
LinearAllocator frameAlloc(1024 * 1024); // 1MB

void ProcessFrame() {
    auto tempData = frameAlloc.Create<FrameData>();
    // ... 处理逻辑
    frameAlloc.Reset(); // 下一帧前重置
}
```

#### 2.2.3 池分配器（PoolAllocator）

**用途**：固定大小对象的频繁分配/释放

```cpp
class PoolAllocator : public IAllocator {
public:
    PoolAllocator(size_t blockSize, size_t blockCount);
    ~PoolAllocator() override;

    void* Allocate(size_t size, size_t alignment = 0) override;
    void Deallocate(void* ptr) override;

    size_t GetBlockSize() const { return blockSize_; }
    size_t GetFreeBlockCount() const;

private:
    void* memoryPool_;
    void* freeList_;
    size_t blockSize_;
    size_t blockCount_;
};
```

#### 2.2.4 堆分配器（HeapAllocator）

**用途**：通用堆分配（封装malloc/free）

```cpp
class HeapAllocator : public IAllocator {
public:
    void* Allocate(size_t size, size_t alignment = alignof(max_align_t)) override;
    void Deallocate(void* ptr) override;

    static HeapAllocator& Get();
};
```

### 2.3 智能指针

#### 2.3.1 UniquePtr

```cpp
namespace mte {

template<typename T>
class UniquePtr {
public:
    UniquePtr() = default;
    explicit UniquePtr(T* ptr);
    ~UniquePtr();

    // 禁止拷贝
    UniquePtr(const UniquePtr&) = delete;
    UniquePtr& operator=(const UniquePtr&) = delete;

    // 移动语义
    UniquePtr(UniquePtr&& other) noexcept;
    UniquePtr& operator=(UniquePtr&& other) noexcept;

    T* Get() const { return ptr_; }
    T& operator*() const { return *ptr_; }
    T* operator->() const { return ptr_; }
    explicit operator bool() const { return ptr_ != nullptr; }

    void Reset(T* ptr = nullptr);
    T* Release();

private:
    T* ptr_ = nullptr;
};

// MakeUnique 工厂函数
template<typename T, typename... Args>
UniquePtr<T> MakeUnique(Args&&... args);

} // namespace mte
```

#### 2.3.2 SharedPtr（简单实现）

```cpp
template<typename T>
class SharedPtr {
public:
    SharedPtr() = default;
    explicit SharedPtr(T* ptr);
    ~SharedPtr();

    SharedPtr(const SharedPtr& other);
    SharedPtr& operator=(const SharedPtr& other);

    T* Get() const { return ptr_; }
    long UseCount() const { return refCount_ ? *refCount_ : 0; }

private:
    void AddRef();
    void Release();

    T* ptr_ = nullptr;
    long* refCount_ = nullptr;
};

template<typename T, typename... Args>
SharedPtr<T> MakeShared(Args&&... args);
```

### 2.4 内存追踪（调试用）

```cpp
namespace mte::memory {

// 内存分配记录
struct AllocationRecord {
    void* ptr;
    size_t size;
    const char* file;
    int line;
    std::chrono::high_resolution_clock::time_point timestamp;
};

class MemoryTracker {
public:
    static MemoryTracker& Instance();

    void Track(void* ptr, size_t size, const char* file, int line);
    void Untrack(void* ptr);

    size_t GetTotalAllocated() const;
    size_t GetAllocationCount() const;
    std::vector<AllocationRecord> GetLeaks() const;

    void DumpToLog();

private:
    std::unordered_map<void*, AllocationRecord> records_;
    std::atomic<size_t> totalAllocated_{0};
};

// 调试宏（开发时启用）
#if defined(MITE_DEBUG)
    #define MITE_TRACK_ALLOC(size) ::mte::memory::MemoryTracker::Instance().Track(...)
    #define MITE_TRACK_FREE(ptr) ::mte::memory::MemoryTracker::Instance().Untrack(...)
#else
    #define MITE_TRACK_ALLOC(size) ((void*)0)
    #define MITE_TRACK_FREE(ptr) ((void*)0)
#endif

} // namespace mte::memory
```

### 2.5 文件组织

```
Memory/
├── CMakeLists.txt
├── IAllocator.hpp
├── LinearAllocator.hpp
├── PoolAllocator.hpp
├── HeapAllocator.hpp
├── UniquePtr.hpp
├── SharedPtr.hpp
├── MemoryTracker.hpp
└── MemoryModule.hpp  # 模块初始化/shutdown
```

---

## 3. Math 子系统

### 3.1 设计目标

- 提供游戏引擎所需的完整数学类型
- 零开销抽象（支持内联、主要操作编译期求值）
- 与GLSL数学库语义兼容
- SIMD优化支持（可选）

### 3.2 向量类型

#### 3.2.1 Vec2

```cpp
namespace mte::math {

struct Vec2 {
    float x = 0.0f;
    float y = 0.0f;

    // 构造函数
    Vec2() = default;
    constexpr Vec2(float x, float y) : x(x), y(y) {}
    explicit Vec2(float scalar) : x(scalar), y(scalar) {}

    // 访问
    float& operator[](size_t idx);
    float operator[](size_t idx) const;

    // 运算
    Vec2 operator+(const Vec2& v) const;
    Vec2 operator-(const Vec2& v) const;
    Vec2 operator*(float s) const;
    Vec2 operator/(float s) const;
    Vec2& operator+=(const Vec2& v);
    Vec2& operator-=(const Vec2& v);
    Vec2& operator*=(float s);
    Vec2& operator/=(float s);

    // 一元
    Vec2 operator-() const;

    // 比较
    bool operator==(const Vec2& v) const;
    bool operator!=(const Vec2& v) const;

    // 长度
    float Length() const;
    float LengthSquared() const;

    // 归一化
    Vec2 Normalized() const;
    void Normalize();

    // 点积、叉积
    float Dot(const Vec2& v) const;
    float Cross(const Vec2& v) const;

    // 线性插值
    static Vec2 Lerp(const Vec2& a, const Vec2& b, float t);

    // 常量
    static const Vec2 Zero;
    static const Vec2 One;
    static const Vec2 UnitX;
    static const Vec2 UnitY;
};

} // namespace mte::math
```

#### 3.2.2 Vec3 / Vec4

结构与Vec2类似，Vec4支持w分量用于齐次坐标。

```cpp
struct Vec3 {
    float x = 0.0f, y = 0.0f, z = 0.0f;

    // ...类似Vec2的接口

    Vec3 Cross(const Vec3& v) const;  // 叉积
    Vec3 Project(const Vec3& normal) const;  // 投影
    Vec3 Reflect(const Vec3& normal) const;  // 反射

    static const Vec3 Zero;
    static const Vec3 One;
    static const Vec3 UnitX;
    static const Vec3 UnitY;
    static const Vec3 UnitZ;
    static const Vec3 Up;
    static const Vec3 Down;
    static const Vec3 Forward;
    static const Vec3 Back;
};

struct Vec4 {
    float x = 0.0f, y = 0.0f, z = 0.0f, w = 0.0f;

    // xyz分量访问（忽略w）
    Vec3 xyz() const;

    static const Vec4 Zero;
    static const Vec4 One;
    static const Vec4 UnitW;
};
```

### 3.3 矩阵类型

#### 3.3.1 Mat3（3x3矩阵）

```cpp
class Mat3 {
public:
    Mat3();
    explicit Mat3(float diagonal);

    // 行列访问
    Vec3& Row(size_t row);
    Vec3 Column(size_t col) const;
    float& At(size_t row, size_t col);
    float At(size_t row, size_t col) const;

    // 运算
    Mat3 operator+(const Mat3& m) const;
    Mat3 operator-(const Mat3& m) const;
    Mat3 operator*(const Mat3& m) const;
    Vec3 operator*(const Vec3& v) const;
    Mat3 operator*(float s) const;

    Mat3& operator+=(const Mat3& m);
    Mat3& operator*=(const Mat3& m);

    // 矩阵操作
    Mat3 Transpose() const;
    Mat3 Inverse() const;
    float Determinant() const;

    // 静态工厂
    static Mat3 Identity();
    static Mat3 Rotate(const Vec3& axis, float radians);
    static Mat3 Scale(const Vec3& scale);

private:
    float m[9];  // 列主序存储
};
```

#### 3.3.2 Mat4（4x4矩阵）

```cpp
class Mat4 {
public:
    Mat4();
    explicit Mat4(float diagonal);
    explicit Mat4(const Mat3& m3);

    // 行列访问
    Vec4& Row(size_t row);
    Vec4 Column(size_t col) const;
    float& At(size_t row, size_t col);
    float At(size_t row, size_t col) const;

    // 运算
    Mat4 operator+(const Mat4& m) const;
    Mat4 operator-(const Mat4& m) const;
    Mat4 operator*(const Mat4& m) const;
    Vec4 operator*(const Vec4& v) const;
    Vec3 operator*(const Vec3& v) const;  // 齐次除法
    Mat4 operator*(float s) const;

    // 变换工厂
    static Mat4 Identity();
    static Mat4 Translate(const Vec3& translation);
    static Mat4 Rotate(const Vec3& axis, float radians);
    static Mat4 RotateX(float radians);
    static Mat4 RotateY(float radians);
    static Mat4 RotateZ(float radians);
    static Mat4 Scale(const Vec3& scale);
    static Mat4 Scale(float uniformScale);

    // 复合变换
    static Mat4 Transform(const Vec3& translation, const Quaternion& rotation, const Vec3& scale);

    // 投影矩阵
    static Mat4 Ortho(float left, float right, float bottom, float top, float near, float far);
    static Mat4 Perspective(float fovY, float aspect, float near, float far);
    static Mat4 PerspectiveFov(float fovY, float width, float height, float near, float far);

    // 视图矩阵
    static Mat4 LookAt(const Vec3& eye, const Vec3& target, const Vec3& up);

    // 矩阵操作
    Mat4 Transpose() const;
    Mat4 Inverse() const;
    Mat4 InverseTransform() const;  // 逆变换（更高效）
    float Determinant() const;

    // 提取子矩阵
    Mat3 ToMat3() const;
    Vec3 GetTranslation() const;
    Quaternion GetRotation() const;
    Vec3 GetScale() const;

private:
    float m[16];  // 列主序存储（与GLSL兼容）
};
```

### 3.4 四元数

```cpp
class Quaternion {
public:
    float x = 0.0f, y = 0.0f, z = 0.0f, w = 1.0f;

    // 构造函数
    Quaternion() = default;
    constexpr Quaternion(float x, float y, float z, float w) : x(x), y(y), z(z), w(w) {}

    // 从轴角创建
    static Quaternion FromAxisAngle(const Vec3& axis, float radians);

    // 从欧拉角创建（XYZ顺序）
    static Quaternion FromEulerAngles(float pitch, float yaw, float roll);

    // 从旋转矩阵创建
    static Quaternion FromRotationMatrix(const Mat3& m);

    // 常用方向创建
    static Quaternion LookRotation(const Vec3& forward, const Vec3& up = Vec3::Up);

    // 运算
    Quaternion operator*(const Quaternion& q) const;  // 旋转组合
    Vec3 operator*(const Vec3& v) const;  // 旋转向量

    Quaternion operator+(const Quaternion& q) const;
    Quaternion operator-(const Quaternion& q) const;
    Quaternion operator-() const;
    Quaternion operator*(float s) const;

    // 长度
    float Length() const;
    float LengthSquared() const;
    void Normalize();
    Quaternion Normalized() const;

    // 共轭、逆
    Quaternion Conjugate() const;
    Quaternion Inverse() const;

    // 插值
    static Quaternion Slerp(const Quaternion& a, const Quaternion& b, float t);
    static Quaternion Nlerp(const Quaternion& a, const Quaternion& b, float t);  // 归一化线性插值

    // 转换
    Vec3 ToEulerAngles() const;  // 返回pitch, yaw, roll
    Mat3 ToRotationMatrix() const;
    Mat4 ToMat4() const;

    // 点积
    float Dot(const Quaternion& q) const;

    // 常量
    static const Quaternion Identity;

private:
    void NormalizeInternal();
};
```

### 3.5 变换（Transform）

```cpp
class Transform {
public:
    Transform();

    // 局部变换
    const Vec3& GetLocalPosition() const { return localPosition_; }
    const Quaternion& GetLocalRotation() const { return localRotation_; }
    const Vec3& GetLocalScale() const { return localScale_; }

    void SetLocalPosition(const Vec3& pos);
    void SetLocalRotation(const Quaternion& rot);
    void SetLocalScale(const Vec3& scale);

    // 世界变换
    Vec3 GetWorldPosition() const;
    Quaternion GetWorldRotation() const;
    Vec3 GetWorldScale() const;

    void SetWorldPosition(const Vec3& pos);
    void SetWorldRotation(const Quaternion& rot);
    void SetWorldScale(const Vec3& scale);

    // 层级关系
    void SetParent(Transform* parent);
    Transform* GetParent() const { return parent_; }
    void DetachFromParent();

    // 矩阵
    Mat4 GetLocalToWorldMatrix() const;
    Mat4 GetWorldToLocalMatrix() const;

    // 变换操作
    void Translate(const Vec3& delta);
    void Rotate(const Quaternion& delta);
    void Rotate(const Vec3& axis, float radians);
    void Scale(const Vec3& factor);

    // 前方/右方/上方
    Vec3 GetForward() const { return localRotation_ * Vec3::Forward; }
    Vec3 GetRight() const { return localRotation_ * Vec3::Right; }
    Vec3 GetUp() const { return localRotation_ * Vec3::Up; }

private:
    Vec3 localPosition_;
    Quaternion localRotation_;
    Vec3 localScale_;

    Transform* parent_ = nullptr;
    std::vector<Transform*> children_;

    mutable bool dirty_ = true;
    mutable Mat4 cachedLocalToWorld_;
};
```

### 3.6 几何类型

#### 3.6.1 Ray（射线）

```cpp
class Ray {
public:
    Ray() = default;
    Ray(const Vec3& origin, const Vec3& direction);

    Vec3 origin;
    Vec3 direction;  // 归一化方向

    // 点在射线上：origin + direction * t
    Vec3 GetPoint(float t) const;

    // 距离到点
    float Distance(const Vec3& point) const;
    float DistanceSquared(const Vec3& point) const;
};
```

#### 3.6.2 AABB（轴对齐包围盒）

```cpp
class AABB {
public:
    AABB();
    AABB(const Vec3& min, const Vec3& max);
    AABB(const Vec3& center, float halfExtent);

    // 边界
    const Vec3& GetMin() const { return min_; }
    const Vec3& GetMax() const { return max_; }
    Vec3 GetCenter() const;
    Vec3 GetExtent() const;  // max - min
    float GetVolume() const;
    float GetSurfaceArea() const;

    // 操作
    void Expand(const Vec3& point);
    void Expand(const AABB& other);
    void Encapsulate(const Vec3& point);
    void Encapsulate(const AABB& other);

    // 包含检测
    bool Contains(const Vec3& point) const;
    bool Contains(const AABB& other) const;
    bool Intersects(const AABB& other) const;

    // 最近点
    Vec3 GetClosestPoint(const Vec3& point) const;

    // 射线相交
    bool Intersects(const Ray& ray, float& t) const;
    bool Intersects(const Ray& ray, float& tMin, float& tMax) const;

    // 静态工厂
    static const AABB Zero;
    static const AABB Infinite;

private:
    Vec3 min_;
    Vec3 max_;
};
```

#### 3.6.3 OBB（有向包围盒）

```cpp
class OBB {
public:
    OBB();
    OBB(const Vec3& center, const Vec3& halfExtents, const Mat3& orientation);

    Vec3 GetCenter() const { return center_; }
    Vec3 GetHalfExtents() const { return halfExtents_; }
    const Mat3& GetOrientation() const { return orientation_; }

    // 获取8个角点
    std::array<Vec3, 8> GetCorners() const;

    // 获取变换矩阵
    Mat4 GetTransformationMatrix() const;

    // AABB转换
    AABB ToAABB() const;

    // 相交检测
    bool Intersects(const Ray& ray, float& t) const;
    bool Intersects(const OBB& other) const;
    bool Intersects(const AABB& aabb) const;

private:
    Vec3 center_;
    Vec3 halfExtents_;
    Mat3 orientation_;  // 列主序旋转矩阵
};
```

#### 3.6.4 Sphere（球体）

```cpp
class Sphere {
public:
    Sphere() = default;
    Sphere(const Vec3& center, float radius);

    Vec3 center;
    float radius = 0.0f;

    // 操作
    void Expand(const Vec3& point);
    void Expand(const Sphere& other);

    // 检测
    bool Contains(const Vec3& point) const;
    bool Intersects(const Sphere& other) const;
    bool Intersects(const Ray& ray, float& t) const;

    // 最近点
    Vec3 GetClosestPoint(const Vec3& point) const;
};
```

#### 3.6.5 Plane（平面）

```cpp
class Plane {
public:
    Plane() = default;
    Plane(const Vec3& normal, float d);  // normal * x + d = 0
    Plane(const Vec3& normal, const Vec3& point);

    Vec3 normal;
    float d = 0.0f;

    // 归一化
    void Normalize();
    Plane Normalized() const;

    // 点到平面距离（签名距离）
    float GetDistance(const Vec3& point) const;
    float GetSignedDistance(const Vec3& point) const;

    // 相交检测
    bool Intersects(const Ray& ray, float& t) const;
    bool Intersects(const Sphere& sphere) const;
    bool Intersects(const AABB& aabb) const;

    // 投影
    Vec3 Project(const Vec3& point) const;

    // 静态工厂
    static Plane FromPoints(const Vec3& a, const Vec3& b, const Vec3& c);
    static Plane FromPointNormal(const Vec3& point, const Vec3& normal);
};
```

### 3.7 数学工具函数

```cpp
namespace mte::math {

// 常量
constexpr float PI = 3.14159265358979323846f;
constexpr float TWO_PI = 6.28318530717958647692f;
constexpr float HALF_PI = 1.57079632679489661923f;
constexpr float INV_PI = 0.31830988618379067154f;
constexpr float DEG_TO_RAD = PI / 180.0f;
constexpr float RAD_TO_DEG = 180.0f / PI;

// 常用函数
template<typename T>
constexpr T Clamp(T value, T min, T max);

template<typename T>
constexpr T Lerp(T a, T b, float t);

template<typename T>
T InverseLerp(T a, T b, T value);

float Sin(float radians);
float Cos(float radians);
float Tan(float radians);
float Asin(float x);
float Acos(float x);
float Atan(float y, float x);
float Atan2(float y, float x);

float Sqrt(float x);
float RSqrt(float x);  // 快速倒数平方根

float Pow(float base, float exponent);
float Exp(float x);
float Log(float x);

float Floor(float x);
float Ceil(float x);
float Round(float x);
float Frac(float x);

float Abs(float x);
float Sign(float x);

float Min(float a, float b);
float Max(float a, float b);
float Min3(float a, float b, float c);
float Max3(float a, float b, float c);

// 随机数（简单实现）
class Random {
public:
    static void Seed(uint32_t seed);
    static float Range(float min, float max);
    static int Range(int min, int max);
    static Vec3 InsideUnitSphere();
    static Vec3 OnUnitSphere();
    static bool CoinFlip();
};

} // namespace mte::math
```

### 3.8 文件组织

```
Math/
├── CMakeLists.txt
├── Vec2.hpp
├── Vec3.hpp
├── Vec4.hpp
├── Mat3.hpp
├── Mat4.hpp
├── Quaternion.hpp
├── Transform.hpp
├── Ray.hpp
├── AABB.hpp
├── OBB.hpp
├── Sphere.hpp
├── Plane.hpp
├── MathConstants.hpp
├── MathFunctions.hpp
└── MathModule.hpp
```

---

## 4. Thread 子系统

### 4.1 设计目标

- 提供跨平台线程抽象
- 高效的线程池实现
- 完整的同步原语
- 原子操作支持

### 4.2 线程基础

#### 4.2.1 线程封装

```cpp
namespace mte::thread {

class Thread {
public:
    using Function = std::function<void()>;
    using Id = std::thread::id;

    Thread();
    explicit Thread(Function func);
    ~Thread();

    // 禁止拷贝
    Thread(const Thread&) = delete;
    Thread& operator=(const Thread&) = delete;

    // 移动
    Thread(Thread&& other) noexcept;
    Thread& operator=(Thread&& other) noexcept;

    // 操作
    void Join();
    void Detach();
    bool Joinable() const;

    Id GetId() const;

    // 静态
    static unsigned int GetHardwareConcurrency();
    static Id GetCurrentId();
    static void Sleep(uint32_t milliseconds);
    static void Yield();

private:
    std::thread thread_;
    Function func_;
};
```

#### 4.2.2 可调用对象适配器

```cpp
namespace mte::thread {

// 将成员函数转换为可调用对象
template<typename Class, typename... Args>
auto Bind(Class* obj, void (Class::*func)(Args...), Args&&... args);

// 示例
class Worker {
public:
    void DoWork(int param) { /* ... */ }
};

Worker* worker = new Worker();
auto task = Bind(worker, &Worker::DoWork, 42);
threadPool.Enqueue(task);

} // namespace mte::thread
```

### 4.3 线程池

#### 4.3.1 接口设计

```cpp
namespace mte::thread {

class ThreadPool {
public:
    explicit ThreadPool(unsigned int threadCount = 0);  // 0 = 自动检测
    ~ThreadPool();

    // 禁止拷贝/移动
    ThreadPool(const ThreadPool&) = delete;
    ThreadPool& operator=(const ThreadPool&) = delete;

    // 任务提交
    template<typename F>
    auto Enqueue(F&& task) -> std::future<std::invoke_result_t<F>>;

    // 批量提交
    template<typename F, typename... Args>
    void EnqueueBatch(F&& func, Args&&... args);

    // 等待所有任务完成
    void Wait();

    // 状态查询
    unsigned int GetThreadCount() const;
    unsigned int GetTaskCount() const;  // 队列中等待的任务数
    bool IsBusy() const;

    // 关闭
    void Shutdown();

private:
    void WorkerThread();

    std::vector<std::thread> workers_;
    std::queue<std::function<void()>> tasks_;

    std::mutex queueMutex_;
    std::condition_variable condition_;
    std::atomic<bool> stop_{false};
    std::atomic<unsigned int> activeTasks_{0};
};
```

#### 4.3.2 使用示例

```cpp
ThreadPool pool(4);  // 4个工作线程

// 提交返回值的任务
auto future = pool.Enqueue([](int x, int y) {
    return x + y;
}, 1, 2);

int result = future.get();  // result = 3

// 提交无返回值任务
pool.Enqueue([]() {
    LOG_INFO("Task executed");
});

// 等待所有任务完成
pool.Wait();
```

### 4.4 同步原语

#### 4.4.1 互斥锁

```cpp
namespace mte::thread {

class Mutex {
public:
    Mutex();
    ~Mutex();

    void Lock();
    bool TryLock();
    void Unlock();

    // RAII 锁
    class ScopedLock {
    public:
        explicit ScopedLock(Mutex& mutex) : mutex_(mutex) { mutex_.Lock(); }
        ~ScopedLock() { mutex_.Unlock(); }
    private:
        Mutex& mutex_;
    };

private:
    std::mutex impl_;
};

// 便捷别名
using Lock = Mutex::ScopedLock;

} // namespace mte::thread
```

#### 4.4.2 读写锁

```cpp
class RWMutex {
public:
    void ReadLock();
    bool TryReadLock();
    void ReadUnlock();

    void WriteLock();
    bool TryWriteLock();
    void WriteUnlock();

    // RAII 锁
    class ReadScopedLock {
    public:
        explicit ReadScopedLock(RWMutex& mutex) : mutex_(mutex) { mutex_.ReadLock(); }
        ~ReadScopedLock() { mutex_.ReadUnlock(); }
    private:
        RWMutex& mutex_;
    };

    class WriteScopedLock {
    public:
        explicit WriteScopedLock(RWMutex& mutex) : mutex_(mutex) { mutex_.WriteLock(); }
        ~WriteScopedLock() { mutex_.WriteUnlock(); }
    private:
        RWMutex& mutex_;
    };

private:
    std::shared_mutex impl_;
};
```

#### 4.4.3 条件变量

```cpp
class ConditionVariable {
public:
    void Wait(Lock& lock);
    template<typename Predicate>
    void Wait(Lock& lock, Predicate pred);

    void NotifyOne();
    void NotifyAll();

private:
    std::condition_variable impl_;
};
```

#### 4.4.4 信号量

```cpp
class Semaphore {
public:
    explicit Semaphore(size_t initialCount = 0);

    void Acquire();        // P操作
    bool TryAcquire();     // 非阻塞
    bool TryAcquireFor(std::chrono::milliseconds timeout);
    void Release();        // V操作
    size_t GetCount() const;

private:
    std::counting_semaphore<> impl_;  // C++20
};
```

#### 4.4.5 屏障

```cpp
class Barrier {
public:
    explicit Barrier(size_t count);

    void Wait();
    void Arrived();  // 单线程到达

private:
    std::barrier<> impl_;
};
```

### 4.5 原子操作

#### 4.5.1 原子计数器

```cpp
namespace mte::thread {

class AtomicCounter {
public:
    explicit AtomicCounter(int64_t initial = 0);

    int64_t Increment();
    int64_t Decrement();
    int64_t Add(int64_t value);
    int64_t Subtract(int64_t value);

    int64_t Load() const;
    void Store(int64_t value);

    int64_t operator++();
    int64_t operator--();
    int64_t operator+=(int64_t rhs);
    int64_t operator-=(int64_t rhs);

    bool CompareExchange(int64_t expected, int64_t desired);

private:
    std::atomic<int64_t> value_;
};

} // namespace mte::thread
```

#### 4.5.2 原子标志

```cpp
class AtomicFlag {
public:
    AtomicFlag() = default;

    void Set();
    void Clear();
    bool Test() const;
    bool TestAndSet();

    void Wait(bool oldValue) const;  // 阻塞直到值改变
    void NotifyOne() const;
    void NotifyAll() const;

private:
    std::atomic_flag flag_;
};
```

### 4.6 并发工具

#### 4.6.1 线程安全队列

```cpp
template<typename T>
class ThreadSafeQueue {
public:
    ThreadSafeQueue() = default;

    void Push(T value);
    bool TryPop(T& value);
    std::optional<T> Pop();  // 阻塞直到有元素

    bool Empty() const;
    size_t Size() const;

    void Clear();

private:
    std::queue<T> queue_;
    mutable std::mutex mutex_;
    std::condition_variable cv_;
};
```

#### 4.6.2 线程局部存储

```cpp
namespace mte::thread {

// 线程局部变量包装
template<typename T>
class ThreadLocal {
public:
    ThreadLocal() = default;
    explicit ThreadLocal(const T& initialValue);

    T& Get();
    const T& Get() const;

    void Reset(const T& value);

    // 线程本地指针（静态）
    template<typename T>
    class LocalPtr {
    public:
        T* Get() const { return ptr_.get(); }
        T* operator->() const { return ptr_.get(); }
        T& operator*() const { return *ptr_.get(); }

        template<typename... Args>
        void Reset(Args&&... args);

    private:
        static thread_local std::unique_ptr<T> ptr_;
    };
};

} // namespace mte::thread
```

### 4.7 文件组织

```
Thread/
├── CMakeLists.txt
├── Thread.hpp
├── ThreadPool.hpp
├── Mutex.hpp
├── RWMutex.hpp
├── ConditionVariable.hpp
├── Semaphore.hpp
├── Barrier.hpp
├── AtomicCounter.hpp
├── AtomicFlag.hpp
├── ThreadSafeQueue.hpp
├── ThreadLocal.hpp
└── ThreadModule.hpp
```

---

## 5. FileSystem 子系统

### 5.1 设计目标

- 跨平台文件路径处理
- 统一文件读写接口
- 虚拟文件系统支持
- 异步IO支持

### 5.2 路径处理

#### 5.2.1 Path类

```cpp
namespace mte::filesystem {

class Path {
public:
    Path();
    Path(const char* path);
    Path(const std::string& path);

    // 字符串转换
    std::string ToString() const;
    const std::string& GetString() const { return path_; }

    // 路径组件
    std::string GetFileName() const;      // "file.txt"
    std::string GetStem() const;          // "file"
    std::string GetExtension() const;     // ".txt"
    std::string GetFolder() const;        // "dir/"
    std::string GetFolderName() const;    // "dir"

    // 路径操作
    Path GetParent() const;
    Path GetAbsolute() const;
    Path GetRelative(const Path& base) const;

    bool IsAbsolute() const;
    bool IsRelative() const;
    bool IsEmpty() const;

    // 拼接
    Path operator/(const Path& other) const;
    Path& operator/=(const Path& other);

    // 修改
    Path& ReplaceExtension(const std::string& ext);
    Path& RemoveExtension();
    Path& MakeAbsolute();

    // 比较
    bool operator==(const Path& other) const;
    bool operator!=(const Path& other) const;

    // 静态工厂
    static Path CurrentDirectory();
    static Path ExecutablePath();
    static Path TemporaryDirectory();

private:
    std::string path_;
};

} // namespace mte::filesystem
```

#### 5.2.2 路径工具函数

```cpp
namespace mte::filesystem {

// 路径操作
bool Exists(const Path& path);
bool IsFile(const Path& path);
bool IsDirectory(const Path& path);
bool IsSymbolicLink(const Path& path);

// 文件操作
size_t GetFileSize(const Path& path);
std::time_t GetLastWriteTime(const Path& path);

// 创建/删除
bool CreateDirectory(const Path& path);
bool CreateDirectories(const Path& path);  // 递归创建
bool Remove(const Path& path);
bool RemoveDirectory(const Path& path);

// 复制/移动
bool CopyFile(const Path& from, const Path& to);
bool MoveFile(const Path& from, const Path& to);

// 遍历
class DirectoryIterator {
public:
    DirectoryIterator(const Path& path);
    DirectoryIterator& operator++();
    bool operator!=(const DirectoryIterator& other) const;
    Path operator*() const;

private:
    std::filesystem::directory_iterator iter_;
};

} // namespace mte::filesystem
```

### 5.3 文件读写

#### 5.3.1 FileHandle

```cpp
namespace mte::filesystem {

enum class FileMode {
    Read,
    Write,
    Append,
    ReadWrite,
    WriteUpdate,
    AppendUpdate
};

enum class FileAccess {
    Read = 1,
    Write = 2,
    ReadWrite = Read | Write
};

class FileHandle {
public:
    FileHandle() = default;
    FileHandle(const Path& path, FileMode mode, FileAccess access = FileAccess::ReadWrite);
    ~FileHandle();

    // 移动语义
    FileHandle(FileHandle&& other) noexcept;
    FileHandle& operator=(FileHandle&& other) noexcept;

    // 状态
    bool IsOpen() const;
    explicit operator bool() const;

    // 读写
    size_t Read(void* buffer, size_t size);
    size_t Write(const void* buffer, size_t size);

    // 指针操作
    void Seek(int64_t offset, int origin);  // SEEK_SET, SEEK_CUR, SEEK_END
    int64_t Tell() const;
    void Flush();

    // 大小
    int64_t GetSize() const;
    void Resize(int64_t newSize);

    // 静态工厂
    static std::optional<FileHandle> Open(const Path& path, FileMode mode);

private:
    std::FILE* file_ = nullptr;
    Path path_;
    FileMode mode_;
};
```

#### 5.3.2 内存映射文件

```cpp
class MemoryMappedFile {
public:
    MemoryMappedFile();
    MemoryMappedFile(const Path& path, bool readOnly = true);
    ~MemoryMappedFile();

    bool Open(const Path& path, bool readOnly = true);
    void Close();

    bool IsOpen() const;

    const void* GetData() const { return data_; }
    void* GetData() { return data_; }
    size_t GetSize() const { return size_; }

    // 范围映射
    const void* GetRange(size_t offset, size_t size) const;
    void* GetRange(size_t offset, size_t size);

private:
    int fd_ = -1;
    void* mapped_ = nullptr;
    void* data_ = nullptr;
    size_t size_ = 0;
    bool readOnly_ = true;
};
```

### 5.4 虚拟文件系统

#### 5.4.1 VFS接口

```cpp
namespace mte::filesystem {

class IVirtualFileSystem {
public:
    virtual ~IVirtualFileSystem() = default;

    // 挂载点管理
    virtual void Mount(const Path& virtualPath, const Path& physicalPath) = 0;
    virtual void Unmount(const Path& virtualPath) = 0;

    // 文件操作（虚拟路径）
    virtual bool Exists(const Path& virtualPath) const = 0;
    virtual bool IsFile(const Path& virtualPath) const = 0;
    virtual bool IsDirectory(const Path& virtualPath) const = 0;
    virtual size_t GetFileSize(const Path& virtualPath) const = 0;

    // 读写
    virtual std::vector<uint8_t> ReadAll(const Path& virtualPath) const = 0;
    virtual std::string ReadAllText(const Path& virtualPath) const = 0;
    virtual bool Write(const Path& virtualPath, const void* data, size_t size) = 0;
    virtual bool WriteText(const Path& virtualPath, const std::string& text) = 0;

    // 遍历
    virtual std::vector<Path> ListDirectory(const Path& virtualPath) const = 0;
};

} // namespace mte::filesystem
```

#### 5.4.2 VFS实现

```cpp
class VirtualFileSystem : public IVirtualFileSystem {
public:
    VirtualFileSystem();

    // 挂载点
    void Mount(const Path& virtualPath, const Path& physicalPath) override;
    void Unmount(const Path& virtualPath) override;

    // 解析虚拟路径到物理路径
    std::optional<Path> Resolve(const Path& virtualPath) const;

    // 文件操作
    bool Exists(const Path& virtualPath) const override;
    bool IsFile(const Path& virtualPath) const override;
    bool IsDirectory(const Path& virtualPath) const override;
    size_t GetFileSize(const Path& virtualPath) const override;

    std::vector<uint8_t> ReadAll(const Path& virtualPath) const override;
    std::string ReadAllText(const Path& virtualPath) const override;
    bool Write(const Path& virtualPath, const void* data, size_t size) override;
    bool WriteText(const Path& virtualPath, const std::string& text) override;

    std::vector<Path> ListDirectory(const Path& virtualPath) const override;

    // 内置包支持
    void MountPackage(const Path& virtualPath, const Path& packageFile);

private:
    struct MountPoint {
        Path virtualPath;
        Path physicalPath;
        bool isPackage = false;
    };

    std::vector<MountPoint> mountPoints_;

    const MountPoint* FindMountPoint(const Path& virtualPath) const;
};
```

#### 5.4.3 使用示例

```cpp
VirtualFileSystem vfs;

// 挂载物理目录
vfs.Mount("/assets", "D:/MiteEngine2/assets");
vfs.Mount("/textures", "D:/MiteEngine2/assets/textures");

// 挂载资源包（只读）
vfs.MountPackage("/levels", "D:/MiteEngine2/data/levels.pkg");

// 统一访问接口
auto data = vfs.ReadAll("/assets/textures/player.png");
auto levelData = vfs.ReadAllText("/levels/level1.json");
```

### 5.5 异步IO

```cpp
namespace mte::filesystem {

class AsyncReader {
public:
    using Callback = std::function<void(std::vector<uint8_t>)>;

    AsyncReader();
    ~AsyncReader();

    // 异步读取
    void ReadAsync(const Path& path, Callback onComplete);

    // 批量异步读取
    void ReadMultipleAsync(const std::vector<Path>& paths,
                          std::function<void(const Path&, std::vector<uint8_t>)> onEach);

    // 等待
    void WaitAll();

private:
    ThreadPool& pool_;
};

class AsyncWriter {
public:
    using Callback = std::function<void(bool success)>;

    AsyncWriter();
    ~AsyncWriter();

    // 异步写入
    void WriteAsync(const Path& path, const void* data, size_t size, Callback onComplete);
    void WriteTextAsync(const Path& path, const std::string& text, Callback onComplete);

    // 等待
    void Flush();

private:
    ThreadPool& pool_;
};

} // namespace mte::filesystem
```

### 5.6 文件组织

```
FileSystem/
├── CMakeLists.txt
├── Path.hpp
├── FileHandle.hpp
├── MemoryMappedFile.hpp
├── IVirtualFileSystem.hpp
├── VirtualFileSystem.hpp
├── AsyncReader.hpp
├── AsyncWriter.hpp
├── FileSystemModule.hpp
└── PathConstants.hpp
```

---

## 6. Logger 子系统

### 6.1 设计目标

- 多级别日志（调试、信息、警告、错误）
- 多输出目标（控制台、文件、回调）
- 可配置的格式化
- 运行时可动态调整级别

### 6.2 日志级别

```cpp
namespace mte::logger {

enum class Level : uint8_t {
    Trace = 0,      // 最详细：函数调用、变量值
    Debug = 1,      // 调试信息：算法步骤、中间状态
    Info = 2,       // 一般信息：重要事件、状态变化
    Warning = 3,   // 警告：潜在问题
    Error = 4,      // 错误：可恢复的错误
    Critical = 5,   // 严重错误：可能导致崩溃
    Off = 6         // 关闭所有日志
};

constexpr const char* LevelToString(Level level) {
    switch (level) {
        case Level::Trace:   return "TRACE";
        case Level::Debug:   return "DEBUG";
        case Level::Info:    return "INFO";
        case Level::Warning: return "WARN";
        case Level::Error:   return "ERROR";
        case Level::Critical: return "CRITICAL";
        default:             return "UNKNOWN";
    }
}

} // namespace mte::logger
```

### 6.3 日志消息

```cpp
namespace mte::logger {

struct Message {
    Level level;
    std::chrono::system_clock::time_point timestamp;
    std::thread::id threadId;

    std::string category;   // 日志分类（如"Render", "Physics"）
    std::string message;     // 日志内容
    std::string file;        // 源文件
    int line;                // 源行号
    std::string function;    // 函数名

    // 格式化后的字符串
    std::string ToString() const;
};

} // namespace mte::logger
```

### 6.4 输出目标

#### 6.4.1 ILogTarget接口

```cpp
namespace mte::logger {

class ILogTarget {
public:
    virtual ~ILogTarget() = default;

    virtual void Log(const Message& message) = 0;
    virtual void Flush() = 0;

    virtual Level GetMinLevel() const = 0;
    virtual void SetMinLevel(Level level) = 0;

    virtual bool IsEnabled() const = 0;
    virtual void SetEnabled(bool enabled) = 0;
};

} // namespace mte::logger
```

#### 6.4.2 ConsoleTarget

```cpp
namespace mte::logger {

enum class ConsoleColor {
    Default,
    Black, Red, Green, Yellow, Blue, Magenta, Cyan, White,
    BrightBlack, BrightRed, BrightGreen, BrightYellow,
    BrightBlue, BrightMagenta, BrightCyan, BrightWhite
};

class ConsoleTarget : public ILogTarget {
public:
    ConsoleTarget(bool useColor = true);

    void Log(const Message& message) override;
    void Flush() override;

    Level GetMinLevel() const override;
    void SetMinLevel(Level level) override;

    bool IsEnabled() const override;
    void SetEnabled(bool enabled) override;

    // 配置
    void SetColorForLevel(Level level, ConsoleColor color);
    void SetOutputStream(std::ostream& stream);

private:
    ConsoleColor GetColorForLevel(Level level) const;
    void SetConsoleColor(ConsoleColor color);

    bool useColor_;
    bool enabled_ = true;
    Level minLevel_ = Level::Trace;
    std::ostream* outputStream_;
    std::unordered_map<Level, ConsoleColor> levelColors_;
};

} // namespace mte::logger
```

#### 6.4.3 FileTarget

```cpp
namespace mte::logger {

class FileTarget : public ILogTarget {
public:
    FileTarget(const Path& filePath, bool append = false);
    ~FileTarget() override;

    void Log(const Message& message) override;
    void Flush() override;

    Level GetMinLevel() const override;
    void SetMinLevel(Level level) override;

    bool IsEnabled() const override;
    void SetEnabled(bool enabled) override;

    // 文件滚动
    void EnableRolling(size_t maxFileSize, size_t maxFiles);
    void SetFilePattern(const std::string& pattern);  // "app_%Y%m%d_%H%M%S.log"

private:
    void RotateIfNeeded();
    void OpenFile();
    void CloseFile();

    Path filePath_;
    std::FILE* file_ = nullptr;
    size_t currentSize_ = 0;
    bool enabled_ = true;
    Level minLevel_ = Level::Trace;

    // 滚动配置
    bool rollingEnabled_ = false;
    size_t maxFileSize_ = 10 * 1024 * 1024;  // 10MB
    size_t maxFiles_ = 5;
    std::string filePattern_;
};

} // namespace mte::logger
```

#### 6.4.4 CallbackTarget

```cpp
namespace mte::logger {

using LogCallback = std::function<void(const Message&)>;

class CallbackTarget : public ILogTarget {
public:
    explicit CallbackTarget(LogCallback callback);

    void Log(const Message& message) override;
    void Flush() override;

    Level GetMinLevel() const override;
    void SetMinLevel(Level level) override;

    bool IsEnabled() const override;
    void SetEnabled(bool enabled) override;

private:
    LogCallback callback_;
    bool enabled_ = true;
    Level minLevel_ = Level::Trace;
};

} // namespace mte::logger
```

### 6.5 日志器

```cpp
namespace mte::logger {

class Logger {
public:
    Logger(const std::string& category);

    // 记录日志
    void Trace(const char* format, ...);
    void Debug(const char* format, ...);
    void Info(const char* format, ...);
    void Warning(const char* format, ...);
    void Error(const char* format, ...);
    void Critical(const char* format, ...);

    // 条件日志
    void TraceIf(bool condition, const char* format, ...);
    void DebugIf(bool condition, const char* format, ...);
    // ...

    // 格式化
    void Log(Level level, const char* format, ...);

    // 类别
    const std::string& GetCategory() const { return category_; }

    // 级别控制
    void SetLevel(Level level);
    Level GetLevel() const;

private:
    void LogVaList(Level level, const char* format, va_list args);

    std::string category_;
    Level level_ = Level::Info;
};

// 全局日志器
Logger& GetDefaultLogger();

} // namespace mte::logger
```

### 6.6 日志系统管理

```cpp
namespace mte::logger {

class LogManager {
public:
    static LogManager& Instance();

    // 初始化/关闭
    void Initialize();
    void Shutdown();

    // 目标管理
    void AddTarget(std::shared_ptr<ILogTarget> target);
    void RemoveTarget(ILogTarget* target);
    void ClearTargets();

    // 全局级别
    void SetGlobalLevel(Level level);
    Level GetGlobalLevel() const;

    // 创建日志器
    Logger& GetLogger(const std::string& category);

    // 全局目标快捷方法
    void AddConsoleTarget(bool useColor = true);
    void AddFileTarget(const Path& filePath);
    void AddCallbackTarget(LogCallback callback);

    // 刷新
    void Flush();

private:
    LogManager() = default;
    ~LogManager() = default;

    std::vector<std::shared_ptr<ILogTarget>> targets_;
    std::unordered_map<std::string, std::unique_ptr<Logger>> loggers_;
    Level globalLevel_ = Level::Info;
};

} // namespace mte::logger
```

### 6.7 使用示例

#### 6.7.1 基础使用

```cpp
// 初始化
LogManager::Instance().Initialize();
LogManager::Instance().AddConsoleTarget(true);
LogManager::Instance().AddFileTarget("logs/app.log");

// 使用
auto& logger = LogManager::Instance().GetLogger("Renderer");
logger.Info("Rendering frame %d", frameNumber);
logger.Warning("Texture %s not found, using default", textureName.c_str());
logger.Error("Failed to create buffer: %s", error.c_str());

// 关闭
LogManager::Instance().Shutdown();
```

#### 6.7.2 便捷宏

```cpp
// 编译时带文件/行号信息的宏
#define MITE_LOG_TRACE(category, ...) \
    ::mte::logger::GetDefaultLogger().Trace(__VA_ARGS__)

#define MITE_LOG_DEBUG(category, ...) \
    ::mte::logger::GetDefaultLogger().Debug(__VA_ARGS__)

#define MITE_LOG_INFO(category, ...) \
    ::mte::logger::GetDefaultLogger().Info(__VA_ARGS__)

#define MITE_LOG_WARNING(category, ...) \
    ::mte::logger::GetDefaultLogger().Warning(__VA_ARGS__)

#define MITE_LOG_ERROR(category, ...) \
    ::mte::logger::GetDefaultLogger().Error(__VA_ARGS__)

#define MITE_LOG_CRITICAL(category, ...) \
    ::mte::logger::GetDefaultLogger().Critical(__VA_ARGS__)

// 使用
MITE_LOG_INFO("Physics", "Player velocity: %.2f", velocity);
```

### 6.8 格式化示例

**默认输出格式**：
```
[2026-03-09 14:30:25.123] [INFO] [Renderer] [main.cpp:42] Render frame 1234
```

**JSON格式（可选）**：
```json
{"timestamp":"2026-03-09T14:30:25.123Z","level":"INFO","category":"Renderer","file":"main.cpp","line":42,"message":"Render frame 1234"}
```

### 6.9 文件组织

```
Logger/
├── CMakeLists.txt
├── Level.hpp
├── Message.hpp
├── ILogTarget.hpp
├── ConsoleTarget.hpp
├── FileTarget.hpp
├── CallbackTarget.hpp
├── Logger.hpp
├── LogManager.hpp
└── LoggerModule.hpp
```

---

## 7. 依赖关系图

```
                    ┌──────────────────────┐
                    │        Core          │
                    │  (Memory, Math,      │
                    │   Thread, FileSystem,│
                    │   Logger)            │
                    └──────────┬───────────┘
                               │
        ┌──────────┬───────────┼───────────┬──────────┐
        ▼          ▼           ▼           ▼          ▼
   ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐
   │Platform │ │  ECS    │ │Rendering│ │Physics  │ │Resources│
   └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘
```

---

## 8. 关键技术决策

| 决策项 | 决策 | 理由 |
|--------|------|------|
| Memory分配器策略 | 三种分配器（线性、池、堆） | 覆盖帧分配、组件池、通用场景 |
| Math库设计 | 自研而非GLM | 零外部依赖、完全控制 |
| 线程池设计 | 静态线程数 | 简化实现、足够使用 |
| VFS设计 | 物理路径+资源包 | 支持DLC、编辑器重定向 |
| 日志系统 | 目标模式 | 灵活扩展输出方式 |

---

## 9. 后续工作

| 任务 | 描述 | 优先级 |
|------|------|--------|
| 内存分配器实现 | 核心分配器开发 | 高 |
| Math库实现 | 向量、矩阵、几何类型 | 高 |
| 线程池实现 | 并发基础 | 中 |
| 日志系统实现 | 调试支持 | 中 |
| VFS实现 | 文件系统抽象 | 中 |
| 单元测试 | 核心功能测试 | 高 |

---

**文档结束**
