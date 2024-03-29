---
title: 【C++】C++引用
date: 2021-11-27
tags: [C++]
cover: https://kakaluoto-hexo-blog.oss-cn-guangzhou.aliyuncs.com/img/202212171659934.webp
mathjax: true
---

## 一. 基本用法

### 1. 例子1

使用引用改变值

```c++
#include <iostream>

#define LOG(x) std::cout<<x<<std::endl

int main() {
    int a = 5;
    int& ref = a;//相当于创建了一个别名
    ref = 2;
    LOG(a);
    return 0;
}
```

输出2

需要注意的是，并不存在真正的引用类型的“变量”ref,ref只是a的一个引用，在编译过后也不会存在ref和a两个变量，ref只存在于源代码中。

### 2. 例子2

给函数传递引用

```c++
#include <iostream>

#define LOG(x) std::cout<<x<<std::endl

void Increment(int& value) {
    value++;
}

int main() {
    int a = 5;
    Increment(a);
    LOG(a);
    return 0;
}
```

输出6

## 二. 需要注意的点

### 1.  一旦声明了一个引用，就不能改变它引用的东西

错误示范

```c++
int a = 5;
int b = 8
int& ref = a;
ref = b;
```

上述代码只是简单地将a的值更改为8，并没有将原本对a的引用改成对b的引用

因此，引用声明的时候必须赋值。

### 2. 与指针的异同
引用能做的指针都能做，指针比引用更为强大，且指针是实际存在的变量。