# 第8章：继承深入 (Inheritance in Depth)

> Stanford CS106L, Fall 2025 -- Rachel Fernandez, Thomas Poimenidis

---

## 目录 (Table of Contents)

1. [类的回顾 (A Recap on Classes)](#1-类的回顾)
2. [继承 (Inheritance)](#2-继承)
3. [虚函数 (Virtual Functions)](#3-虚函数)
4. [纯虚函数与抽象类](#4-纯虚函数与抽象类)
5. [组合 vs 继承](#5-组合-vs-继承)
6. [本章回顾](#6-本章回顾)
7. [补充知识点](#7-补充知识点)

---

## 1. 类的回顾

### 1.1 什么是类？

一个类代表一种**抽象** -- 它可以建模真实世界的对象、概念，或任何你想组织为数据和行为的实体。类将数据和操作数据的方法打包在一起。

类比现实生活中的概念：
- Car（汽车）
- Engine（引擎）
- Video Game Character（游戏角色）
- Vector（向量）
- Graph（图）
- Book（书）
- Dog（狗）

### 1.2 Point 类示例

```cpp
// Point.h (头文件 -- 包含接口和声明)
class Point {
public:
    Point(int x, int y);    // 构造函数：初始化对象
    ~Point();               // 析构函数：清理对象（通常不需要）
    int getX();             // 获取器
    int getY();
    void setX(int x);       // 设置器
    void setY(int y);

private:                    // 私有成员：只有类内部可见（实现细节）
    int x;
    int y;
};

// Point.cpp (源文件 -- 包含实现和定义)
#include "Point.h"

Point::Point(int x, int y) : x(x), y(y) {}

int Point::getX() { return x; }
int Point::getY() { return y; }
```

### 1.3 Python vs C++ 类

Python 类：

```python
class Point:
    def __init__(self, x, y):
        self._x = x
        self._y = y

    def getX(self):
        return self._x

    def getY(self):
        return self._y
```

C++ 类：

```cpp
class Point {
private:
    int x;
    int y;
public:
    Point(int x, int y) : x{x}, y{y} {}
    int getX() { return x; }
    int getY() { return y; }
};
```

### 1.4 内存布局对比

**Python 的内存布局**（开销很大）：

```
p = Point(1, 2)

p --> Refcount = 1
      type = Point
      __dict__  --> refcount = 1
                    type = dict
                    size = 2
                    keys   --> refcount=1, type=str, data="_x" / "_y"
                    values --> refcount=1, type=int, digits=[1] / [2]
```

Python 存储了大量额外信息：
- refcount（引用计数）用于垃圾回收
- type（类型信息）用于运行时类型检查
- __dict__（属性字典）用于动态属性访问
- 每个属性和值也有独立的类型元数据

**C++ 的内存布局**（简洁高效）：

```cpp
Point p {1, 2};

p --> int x = 1
      int y = 2
```

C++ 只在对象中存储数据本身！编译器在**编译时**完成所有类型检查。

**总结**：C++ 和 Python 内存布局的本质区别是 C++ 高性能的重要原因之一。C++ 不存储运行时类型信息（除非使用虚函数），编译器在编译时就确定了一切。

### 1.5 函数存储在哪里？

函数**不存储在对象内部**，而是分别存储在程序的文本段（text/instructions区域）。

在 C++ 中，`this` 指针被隐式传递给每个成员函数：

```cpp
int Point::getX() {
    return this->x;
}

Point p {1, 2};
int x = p.getX();

// 编译器实际上将上述代码转换如下：
// int Point_getX(Point* this) { return this->x; }
// Point p {1, 2};
// int x = Point_getX(&p);
```

### 1.6 `this` 指针的重要性

```cpp
// ✅ 这两个是等价的（编译器隐式添加 this->）
int Point::getX() {          int Point::getX() {
    return x;                    return this->x;
}                            }

// ❌ 这两个不等价！（局部变量遮蔽了成员变量）
void Point::setX(int x) {    void Point::setX(int x) {
    x = x;                        this->x = x;
}                            }
```

`this` 的类型：对于非const成员函数是 `Point*`，对于const成员函数是 `const Point*`。

---

## 2. 继承 (Inheritance)

### 2.1 继承的动机

继承是一种让一个类从另一个类获取属性的机制。

现实世界中的继承关系：

- **汽车**：丰田凯美瑞、本田思域、福特野马 -- 都拥有引擎、轮子、方向盘
- **形状**：盒子、球体、圆锥、环 -- 都有体积、表面积
- **流**：`std::ifstream` 是一个 `std::istream` 是一个 `std::ios`

### 2.2 游戏对象建模

假设我们在写一个类似Fortnite的游戏，每个对象都有位置、碰撞盒、更新和渲染方法：

```cpp
class Player {
    double x, y, z;        HitBox hitbox;
    double hitpoints;
public:
    void damage(double hp); void update(); void render();
};

class Projectile {
    double x, y, z;        HitBox hitbox;
    double vx, vy, vz;     // 速度分量
public:
    void update();          void render();
};

class Weapon {
    double x, y, z;        HitBox hitbox;
    size_t ammo;
public:
    void fire();            void update(); void render();
};

class Tree {
    double x, y, z;        HitBox hitbox;
public:
    void update();          void render();
};

class NPC {
    double x, y, z;        HitBox hitbox;
    double hitpoints;
public:
    void damage(double hp); void update(); void render();
};
```

**大量重复代码！** 而且修改非常痛苦 -- 如果我们需要为每个对象添加一个碰撞检测方法 `overlapsWith`，必须在每个类中实现多个重载版本。

### 2.3 提取公共基类

```cpp
// 所有游戏对象的公共基类
class Entity {
    double x, y, z;
    HitBox hitbox;
public:
    void update();
    void render();
};
```

### 2.4 使用继承消除冗余

```cpp
class Player : public Entity {
    double hitpoints;
public:
    void damage(double hp);
};

class Projectile : public Entity {
    double vx, vy, vz;
};

class Weapon : public Entity {
    size_t ammo;
public:
    void fire();
};

class Tree : public Entity {};

class NPC : public Entity {
    double hitpoints;
public:
    void damage(double hp);
};
```

### 2.5 更深的继承层级

可以进一步抽象：

```cpp
class Entity {
    double x, y, z;
    HitBox hitbox;
public:
    void update();
    void render();
};

class Actor : public Entity {
    double hitpoints;
public:
    void damage(double hp);
};

class Player : public Actor {};
class NPC : public Actor {};

class Projectile : public Entity {
    double vx, vy, vz;
};

class Weapon : public Entity {
    size_t ammo;
public:
    void fire();
};

class Tree : public Entity {};
```

继承树定义了 **is-a** 关系：
- "Weapon 是一个 Entity"
- "NPC 是一个 Actor，也间接是一个 Entity"

### 2.6 访问修饰符与继承

**关键点**：类默认是私有继承的！

```cpp
// 默认私有继承（通常不是你想要的）
class Player : /* private */ Entity {
    // 私有继承:
    // - Entity的public成员在Player中变为private
    // - 外部无法访问overlapsWith
};

// 公开继承（正确建模 is-a 关系）
class Player : public Entity {
    // 公开继承:
    // - Entity的public成员在Player中仍为public
    // - 外部可以访问overlapsWith
};
```

### 2.7 访问修饰符总结表

| 继承方式 | 基类private成员 | 基类public成员 | 基类protected成员 |
|---------|----------------|---------------|------------------|
| `private` (默认) | 不可访问 | 变为private | 变为private |
| `protected` | 不可访问 | 变为protected | 变为protected |
| `public` | 不可访问 | 保持public | 保持protected |

### 2.8 `protected` 访问修饰符

`protected` 成员对子类可见，但对外部不可见：

```cpp
class Entity {
protected:              // 子类可以访问这些
    double x, y, z;
    HitBox hitbox;
public:
    void update();
    void render();
};

class Projectile : public Entity {
private:
    double vx, vy, vz;
public:
    void move() {
        x += vx;        // ✅ 子类可以访问 protected 成员
        y += vy;
        z += vz;
    }
};

// 外部代码
Projectile p;
// p.x = 5;            // ❌ 外部不能访问 protected 成员
```

### 2.9 对象切片 (Object Slicing)

**重要陷阱**：当将派生类赋值给基类变量时，会发生对象切片：

```cpp
// 内存布局
class Entity {
    double x, y, z;
    HitBox hitbox;
};

class Projectile : public Entity {
    double vx, vy, vz;  // 这些在赋值给Entity时会被切掉！
};
```

```cpp
// 错误示例 -- 对象切片
int main() {
    std::vector<Entity> entities { Player(), Tree(), Projectile() };
    for (auto& entity : entities) {
        entity.update();   // 始终调用 Entity::update()！
        entity.render();   // 始终调用 Entity::render()！
    }
}
// 每个元素都是 Entity，派生类的额外数据被切掉了！
```

**解决方案：使用指针**

```cpp
int main() {
    Player p; Tree t; Projectile b;
    std::vector<Entity*> entities { &p, &t, &b };
    for (auto& ent : entities) {
        ent->update();     // 现在保留了完整的子类信息
        ent->render();
    }
}
```

指针通过避免拷贝来保留子类的完整细节。

---

## 3. 虚函数 (Virtual Functions)

### 3.1 问题：编译器不知道该调用哪个方法

```cpp
Entity* entity = entities[0];
entity->update();  // 应该调用哪个 update()？

// 我们有的是：Entity::update()、Player::update()、
//            Projectile::update()、NPC::update()...
```

给定一个 `Entity*`，编译器如何知道调用哪个方法？

- 如果 `entity` 指向 `Player`，应该调用 `Player::update()`
- 如果指向 `Projectile`，应该调用 `Projectile::update()`

但 `Entity*` 本身不包含类型信息 -- 编译器默认假设它指向 `Entity`。

### 3.2 编译时类型 vs 运行时类型

使用 `Entity*` 有一个代价：我们"忘记"了对象的实际类型。

- **编译时类型**：`Entity*` -- 编译器将其视为 Entity
- **运行时类型**：可能是 Entity 或任何子类（Projectile、Player等）

我们需要的是一种**动态分派 (Dynamic Dispatch)**机制：根据对象的运行时（动态）类型来调用（分派）不同的方法。

### 3.3 `virtual` 关键字

将函数标记为 `virtual` 即可启用动态分派：

```cpp
class Entity {
public:
    virtual void update() {}     // 虚函数
    virtual void render() {}
};

class Projectile : public Entity {
public:
    void update() override {}    // override 不是必需的，但强烈推荐！
};                               // 编译器会检查是否真的覆写了虚函数
```

**`override` 关键字的好处**：
- 提升可读性（明确表示这是覆写）
- 编译器会检查是否真的覆写了基类的虚方法（防止拼写错误或签名不匹配）

### 3.4 虚函数的工作原理：虚表 (vtable)

添加 `virtual` 会在每个对象中添加一个**虚指针 (vpointer)**，指向**虚表 (vtable)**。虚表记录了每个虚方法对应调用哪个函数。

```
Entity* entity --> Projectile 对象
                   +------------------+
                   | double x         |
                   | double y         |
                   | double z         |
                   | HitBox hitbox    |
                   | void* vpointer  ----> vtable:
                   | double vx        |    update --> Projectile::update()
                   | double vy        |    render --> Entity::render()
                   | double vz        |
                   +------------------+
```

### 3.5 虚函数 vs Python 的动态类型

两者都存储了类型相关信息：

- **Python**：在每个对象中存储 refcount、type、__dict__ 等大量元数据，支持动态类型和运行时类型检查
- **C++ 虚函数**：只添加一个 vpointer（8字节，在64位系统上），额外开销极小。当你需要多态时才选择使用。

### 3.6 虚函数的权衡

| 优点 | 缺点 |
|------|------|
| 动态分派 -- 运行时根据实际类型调用正确方法 | 增加内存布局大小（每个对象多一个vpointer） |
| 灵活的可扩展设计 | 调用方法时需要查虚表（间接调用，比直接调用稍慢） |
| C++的关键OOP特性 | 编译器无法内联虚函数调用 |

在量化金融等对纳秒级延迟敏感的领域中，通常不使用虚函数！

---

## 4. 纯虚函数与抽象类

### 4.1 纯虚函数语法

```cpp
class Entity {
public:
    virtual void update() = 0;   // 纯虚函数：= 0 代替实现
    virtual void render() = 0;
};
```

### 4.2 抽象类

- 包含一个或多个纯虚函数的类是**抽象类**，**不能实例化**
- 覆写**所有**纯虚函数后，类才变为**具体类**

```cpp
class Entity {                            // 抽象类
public:
    virtual void update() = 0;
    virtual void render() = 0;
};

class Projectile : public Entity {        // 具体类（覆写了所有纯虚函数）
public:
    void update() override {};
    void render() override {};
};

Entity e;        // ❌ Entity是抽象类，不能实例化！
Projectile p;    // ✅ Projectile覆盖了所有纯虚函数，是具体类
```

### 4.3 何时使用纯虚函数

当**没有明确的默认实现**时使用纯虚函数：

```cpp
class Shape {
public:
    virtual double volume() = 0;  // 形状的体积没有通用默认值
};
// 让 Box、Sphere、Cone 等子类各自定义 volume()
```

---

## 5. 组合 vs 继承

### 5.1 继承可能失控

过大的继承树往往更慢、更难推理：
- 在现代游戏引擎中，为每个不同对象类型创建子类的做法已不常见
- 组合通常更灵活、更合理

### 5.2 "A car is an engine" vs "A car has an engine"

```cpp
// ❌ 继承：逻辑上不合理
class Car : public Engine
         , public SteeringWheel
         , public Brakes {
    // 这看起来不太对...
};

// ✅ 组合：更自然的建模
class Car {
    Engine engine;
    SteeringWheel wheel;
    Brakes brakes;
};
```

### 5.3 组合 + 继承 = 最佳实践

结合两者的优势：

```cpp
class Car {
    Engine* engine;           // 组合 + 多态
    SteeringWheel* wheel;
    Brakes* brakes;
};

// Engine 层级可以使用继承
class Engine {};
class CombustionEngine : public Engine {};
class GasEngine : public CombustionEngine {};
class DieselEngine : public CombustionEngine {};
class ElectricEngine : public Engine {};
```

这种技术在C++中的一个著名应用是 **PIMPL 惯用法 (Pointer to IMPLementation)**。

### 5.4 设计原则：优先使用组合而非继承

继承是强大的工具，但有时候组合更有意义。两者的适用场景：

| 场景 | 推荐方案 |
|------|---------|
| 明确的 is-a 关系 (Dog is an Animal) | 继承 |
| 需要多态行为 | 继承 + 虚函数 |
| has-a 关系 (Car has an Engine) | 组合 |
| 需要在运行时更换行为 | 组合（策略模式） |
| 避免深继承树 | 组合 |

---

## 6. 本章回顾

| 概念 | 要点 |
|------|------|
| **类回顾** | 类打包数据和行为；C++内存布局极简（只存数据）；`this` 指针由编译器隐式传递 |
| **继承** | 一个类从另一个类继承属性；减少代码冗余；默认私有继承，通常应使用 public 继承 |
| **虚函数** | `virtual` 启用动态分派；由虚表 (vtable) 实现；C++需要显式选择（不像Java/Python默认多态） |
| **纯虚函数** | `= 0` 声明纯虚函数；抽象类不能实例化；适合没有默认实现的情况 |
| **组合 vs 继承** | 优先使用组合；is-a 用继承，has-a 用组合；两者结合效果最佳 |

---

## 7. 补充知识点

### 7.1 `final` 关键字 (C++11)

```cpp
class Base final {           // 此类不能被继承
    // ...
};

class Derived : public Base { // ❌ 编译错误
};

class Animal {
public:
    virtual void speak() final;  // 此虚函数不能在子类中被覆写
};

class Dog : public Animal {
public:
    void speak() override;   // ❌ 编译错误
};
```

### 7.2 协变返回类型 (Covariant Return Types)

派生类覆写虚函数时可以返回更具体的类型：

```cpp
class Base {
public:
    virtual Base* clone() const;
};

class Derived : public Base {
public:
    Derived* clone() const override;  // ✅ 返回更具体的类型
};
```

### 7.3 运行时类型识别 (RTTI)

```cpp
// dynamic_cast -- 安全的向下转型
Entity* entity = GetEntityFromSomewhere();
if (Player* player = dynamic_cast<Player*>(entity)) {
    player->damage(10);  // 安全地当作 Player 使用
}

// typeid -- 获取运行时类型信息
if (typeid(*entity) == typeid(Player)) {
    // entity 指向的是 Player
}
```

**注意**：`dynamic_cast` 需要虚函数表（至少一个虚函数），有一定运行时开销。过度使用可能意味着设计需要改进。

### 7.4 多重继承

C++支持多重继承，但需谨慎使用：

```cpp
class FlyingAnimal {
public:
    virtual void fly() = 0;
};

class SwimmingAnimal {
public:
    virtual void swim() = 0;
};

class Duck : public FlyingAnimal, public SwimmingAnimal {
public:
    void fly() override { /* ... */ }
    void swim() override { /* ... */ }
};
```

多重继承可能导致**菱形问题**（见第7章），需使用虚继承解决。

### 7.5 接口类 (Interface Classes)

在C++中，可以用只有纯虚函数的抽象类模拟接口：

```cpp
class IPrintable {
public:
    virtual void print() const = 0;
    virtual ~IPrintable() = default;
};

class ISerializable {
public:
    virtual std::string serialize() const = 0;
    virtual ~ISerializable() = default;
};

// 多重继承接口是常见的C++模式
class Document : public IPrintable, public ISerializable {
public:
    void print() const override { /* ... */ }
    std::string serialize() const override { /* ... */ }
};
```

### 7.6 非虚接口 (NVI) 惯用法

将虚函数设为私有，通过公共的非虚函数调用：

```cpp
class GameObject {
public:
    void update() {                    // 非虚接口
        doPreUpdate();
        doUpdate();                    // 调用虚函数
        doPostUpdate();
    }
private:
    virtual void doUpdate() = 0;       // 子类覆写这个
    void doPreUpdate() { /* 公共逻辑 */ }
    void doPostUpdate() { /* 公共逻辑 */ }
};
```

这样可以确保基类的前置/后置逻辑始终被执行。

### 7.7 CRTP (Curiously Recurring Template Pattern)

一种编译时多态的替代方案，无需虚函数：

```cpp
template <typename Derived>
class Base {
public:
    void interface() {
        static_cast<Derived*>(this)->implementation();
    }
};

class Derived : public Base<Derived> {
public:
    void implementation() {
        // 具体实现
    }
};
```

CRTP 避免了虚函数开销，完全在编译时解析。
