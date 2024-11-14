---
title: 【C++】C++类
date: 2024-11-14
tags: [C++]
cover: https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/2021_10_19_6.webp
mathjax: true
---

# C++类

## C++的类长什么样？

```C++
#include <iostream>

class Player
{
    int x, y;
    int speed;
};

int main() {
    Player player;
    player.x = 0;
    player.y = 0;
    player.speed = 0;
    return 0;
}

```

以上Class就构成了C++最简单的一个类，但是如果编译就会发现报错，这是因为C++类成员变量不经过任何声明默认就是私有的。

```
D:\MyCppProject\untitled\main.cpp:12:12: error: 'int Player::x' is private within this context
   12 |     player.x = 0;
```

当定义成如下形式，就可以正常访问

```C++
class Player
{
public:
    int x, y;
    int speed;

};
```

加入方法的形式

```C++
class Player
{
public:
    int x, y;
    int speed;

    void Move(int xa, int ya) {
        x += xa * speed;
        y += ya * speed;
    }

};
```

## C++类与结构体的比较

C++类成员方法和成员变量默认都是private的，但是在struct中它们是public的

```C++
#include <iostream>

class Player
{
    int x, y;
    int speed;

    void move(int xa, int ya) {
        x += xa * speed;
        y += ya * speed;
    }

};

int main() {
    Player player;
    player.x = 0;
    player.y = 0;
    player.speed = 0;
    player.move(1,1);
    return 0;
}
```

当我们去掉public的声明形式，编译不会通过，因为x,y以及move函数都是不可见无法访问的，但是当你将类换成结构体就可以访问了

```C++
#include <iostream>

struct Player
{

    int x, y;
    int speed;

    void move(int xa, int ya) {
        x += xa * speed;
        y += ya * speed;
    }

};

int main() {
    Player player;
    player.x = 0;
    player.y = 0;
    player.speed = 0;
    player.move(1,1);
    return 0;
}
```

你甚至可以在struct里进行private声明

```C++
struct Player
{
private:
    int x, y;
    int speed;

    void move(int xa, int ya) {
        x += xa * speed;
        y += ya * speed;
    }

};
```

这样就又回到了最初Class里定义成priavte的状态，即外部无法访问。

从技术上来说Class和Struc的区别就这些了，除了可见性之外，Class支持的特性Struct也都支持(比如继承)，但当我们实际使用时，更推荐当仅仅需要将不同变量进行组合时使用Struct，需要涉及到方法时使用Class。

## 麻雀虽小五脏俱全的类

这是一个小demo，之后的学习将以这个demo为基础引入各种概念进行改进

```C++
#include <iostream>

class Log
{
public:
    const int LogLevelError = 0;
    const int LogLevelWarning = 1;
    const int LogLevelInfo = 2;
private:
    int m_LogLevel = LogLevelInfo;
public:
    void SetLevel(int level) {
        m_LogLevel = level;
    }

    void Error(const char *message) {
        if (LogLevelError <= m_LogLevel) {
            std::cout << "[Error]: " << message << std::endl;
        }
    }

    void Warn(const char *message) {
        if (LogLevelWarning <= m_LogLevel) {
            std::cout << "[WARNING]: " << message << std::endl;
        }
    }

    void Info(const char *message) {
        if (LogLevelInfo <= m_LogLevel) {
            std::cout << "[Info]: " << message << std::endl;
        }
    }
};

int main() {
    Log log;
    log.SetLevel(log.LogLevelWarning);
    log.Warn("hello");
    log.Error("hello");
    log.Info("hello");
    return 0;
}
```





















