---
tags:
  - 嵌入式/编程语言/CPP
number headings: " first-level l, start-at 1, max 6, l.l, auto, contents"
aliases:
---

# 1 场景一：多态代替 Switch（以图形绘制为例）

以绘制图形为例，如果我们使用 switch 来实现相关功能，当我们每增加一种新的数据类型（如新图形），就要去修改所有包含 `switch-case` 的函数时，你就违反了 [[六大设计原则-开闭原则|开闭原则]]。

# 2 反面教材（面向过程的 Switch）

这种写法很常见，但隐患巨大。每次加一个 `Triangle`，我们都得去 `grep` 全局搜索 `case SQUARE:`，然后在后面补代码。漏改一处，Bug 就来了。

```cpp
#include <iostream>
#include <vector>

// 定义类型枚举
enum ShapeType { CIRCLE, SQUARE };

struct Shape {
    ShapeType type;
};

// 典型的坏代码
void draw_all_shapes(const std::vector<Shape*>& shapes) {
    for (auto s : shapes) {
        switch (s->type) {
            case CIRCLE:
                std::cout << "Drawing Circle" << std::endl;
                break;
            case SQUARE:
                std::cout << "Drawing Square" << std::endl;
                break;
            // 如果加了 Triangle，这里没改，编译器可能不会报错，但逻辑全错！
        }
    }
}
```

# 3 推荐写法（面向对象的多态）

我们可以利用**虚函数**，将我是谁，我该怎么画的逻辑内聚到对象内部。基类提供统一接口，具体实现下放给子类。

```
#include <iostream>
#include <vector>
#include <memory> // 使用智能指针管理内存

// 抽象基类
class Shape {
public:
    virtual ~Shape() {} 
    virtual void draw() const = 0; // 纯虚函数，强制子类实现
};

class Circle : public Shape {
public:
    void draw() const override { 
        std::cout << "Drawing Circle (Polymorphism)" << std::endl;
    }
};

class Square : public Shape {
public:
    void draw() const override {
        std::cout << "Drawing Square (Polymorphism)" << std::endl;
    }
};

// 程序员视角：
// 无论加多少种图形，这个函数永远不用改！符合“对扩展开放，对修改封闭”。
void draw_all_shapes(const std::vector<std::unique_ptr<Shape>>& shapes) {
    for (const auto& s : shapes) {
        s->draw(); // 编译器会自动分派到对应的 draw 实现
    }
}

int main() {
    std::vector<std::unique_ptr<Shape>> shapes;
    shapes.push_back(std::make_unique<Circle>());
    shapes.push_back(std::make_unique<Square>());

    draw_all_shapes(shapes);
    return 0;
}
```
