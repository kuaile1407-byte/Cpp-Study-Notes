---
tags:
  - 嵌入式/编程语言/CPP
  - 嵌入式/编程语言/C语言
number headings: " first-level l, start-at 1, max 6, l.l, auto, contents"
aliases:
---

# 1 说明

在开发过程中，如果状态之间的跳转非常复杂（例如：只有在奔跑时按下跳跃才能变成跳远，否则是原地跳）。用 `switch` 嵌套 `if-else`，代码会变成难以维护的状态爆炸,。

# 2 反面教材（状态机里的 Switch 地狱）

所有的逻辑都塞在一个函数里，变量（如 `speed`）不仅被奔跑状态修改，可能还会被跳跃状态意外修改，耦合度极高。

```cpp
enum State { IDLE, RUNNING, JUMPING };

void handleInput(State& currentState, char input) {
    switch (currentState) {
        case IDLE:
            if (input == 'r') {
                currentState = RUNNING;
                std::cout << "开始奔跑" << std::endl;
            } else if (input == 'j') {
                currentState = JUMPING; // 原地跳
                std::cout << "原地起跳" << std::endl;
            }
            break;
        case RUNNING:
            if (input == 'j') {
                currentState = JUMPING; // 跑动跳
                std::cout << "超级跳跃！" << std::endl;
            } else if (input == 's') {
                currentState = IDLE;
                std::cout << "停下" << std::endl;
            }
            break;
        // … 随着状态增加，这个 switch 会膨胀到几千行
    }
}
```

# 3 推荐写法（状态模式 State Pattern）

## 3.1 C++ 版本

我们将每个状态封装成一个**类**。`Context`（环境类，如 Hero）持有当前状态的指针。状态的流转逻辑分散在各个状态类中，而不是堆积在一个大 `switch` 里,。

```
#include <iostream>
#include <memory>

class Hero; // 前置声明

// 状态基类
class HeroState {
public:
    virtual ~HeroState() {}
    virtual void handleInput(Hero& hero, char input) = 0;
};

// 2. 环境类 (Context)
class Hero {
    std::unique_ptr<HeroState> state; // 持有当前状态
public:
    Hero(); // 需要在 cpp 中实现 

    void setState(std::unique_ptr<HeroState> newState) {
        state = std::move(newState);
    }

    void handleInput(char input) {
        // 核心：将请求委托给当前状态对象处理
        state->handleInput(*this, input);
    }
};

// 具体状态实现
// -------------------------
class RunningState : public HeroState {
public:
    void handleInput(Hero& hero, char input) override;
};

class IdleState : public HeroState {
public:
    void handleInput(Hero& hero, char input) override {
        if (input == 'r') {
            std::cout << "站立 -> 开始奔跑" << std::endl;
            // 切换状态
            hero.setState(std::make_unique<RunningState>());
        } else {
            std::cout << "站立中…无反应" << std::endl;
        }
    }
};

// RunningState 的实现 (需要在 IdleState 定义之后)
void RunningState::handleInput(Hero& hero, char input) {
    if (input == 's') {
        std::cout << "奔跑 -> 紧急刹车" << std::endl;
        hero.setState(std::make_unique<IdleState>());
    } else {
        std::cout << "跑啊跑…" << std::endl;
    }
}

// Hero 构造函数初始化初始状态
Hero::Hero() : state(std::make_unique<IdleState>()) {}

int main() {
    Hero hero;
    hero.handleInput('r'); // 输出：站立 -> 开始奔跑
    hero.handleInput('s'); // 输出：奔跑 -> 紧急刹车
    return 0;
}
```

## 3.2 C 语言版本

我们利用 C 语言的结构体也可以实现类似的功能：

```cpp
#include <stdio.h>
#include <stdlib.h>

// 前置声明英雄结构体，因为状态函数需要它作为参数
struct Hero;

// 状态基类（用结构体模拟）
typedef struct HeroState {
    // 函数指针，相当于 C++ 中的虚函数 handleInput
    void (*handleInput)(struct Hero* hero, char input);
} HeroState;

// 环境类/上下文 (Context) - 我们的英雄
typedef struct Hero {
    const HeroState* currentState; // 持有当前状态的指针（使用const指针指向不变的状态对象）
} Hero;

// 状态函数的原型声明
void idleState_handleInput(Hero* hero, char input);
void runningState_handleInput(Hero* hero, char input);

// 全局的、预定义的状态对象（通常是const，因为其行为是固定的）
// 站立状态
const HeroState g_idleState = { .handleInput = idleState_handleInput };
// 奔跑状态
const HeroState g_runningState = { .handleInput = runningState_handleInput };

// 英雄的初始化函数
void Hero_Init(Hero* hero) {
    hero->currentState = &g_idleState; // 初始状态为站立
}

// 英雄处理输入的函数：将请求委托给当前状态
void Hero_handleInput(Hero* hero, char input) {
    if (hero->currentState && hero->currentState->handleInput) {
        hero->currentState->handleInput(hero, input);
    }
}

// 英雄设置状态的函数
void Hero_setState(Hero* hero, const HeroState* newState) {
    if (newState) {
        hero->currentState = newState;
    }
}

// 具体状态实现
// -------------------------
// 站立状态的输入处理
void idleState_handleInput(Hero* hero, char input) {
    if (input == 'r') {
        printf("站立 -> 开始奔跑\n");
        // 切换状态：将英雄的当前状态指针指向奔跑状态对象
        Hero_setState(hero, &g_runningState);
    } else {
        printf("站立中…无反应\n");
    }
}

// 奔跑状态的输入处理
void runningState_handleInput(Hero* hero, char input) {
    if (input == 's') {
        printf("奔跑 -> 紧急刹车\n");
        // 切换状态：将英雄的当前状态指针指向站立状态对象
        Hero_setState(hero, &g_idleState);
    } else {
        printf("跑啊跑…\n");
    }
}

// 主函数，演示用法
int main() {
    Hero hero;
    Hero_Init(&hero); // 初始化英雄，状态为 Idle

    Hero_handleInput(&hero, 'r'); // 输出：站立 -> 开始奔跑 (状态切换为 Running)
    Hero_handleInput(&hero, 's'); // 输出：奔跑 -> 紧急刹车 (状态切换回 Idle)
    Hero_handleInput(&hero, 'x'); // 输出：跑啊跑… (在奔跑状态下按其他键)

    return 0;
}
```
