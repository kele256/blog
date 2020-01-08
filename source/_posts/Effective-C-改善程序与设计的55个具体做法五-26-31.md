---
title: Effective-C-改善程序与设计的55个具体做法(五)(26~31)
date: 2020-01-05 22:47:45
tags:
- Effective C++
categories:
- C++
---

#### 五. 实现

##### 26. 尽可能延后变量定义式的出现时间

尽量避免定义不使用的变量。或许你会认为，你不可能定义一个不使用的变量，但话不要说得太早！考虑如下的函数，它计算通行密码的加密版本而后返回，前提是密码够长。如果密码太短，函数会丢出一个异常，类型为logic_error：

```C++
std::string encryptPassword(const std::string& password) {
    using namespace std;
    string encrypted;
    if (password.lenth() < MinimumPasswordLength) {
        throw logic_error("Password is too short");
    }
    ... // 必要动作，将一个加密后的密码置入变量encrypted内。
    return encrypted;
}
```

对象encrypted在此函数中并非完全未被使用，但如果有个异常被抛出，它就真的没被使用。最好延后它的定义式，直到确实需要他：

```C++
std::string encryptPassword(const std::string& password) {
    using namespace std;
    if (password.length() < MinimumPasswordLength) {
        throw logic_error("Password is too short");
    }
    string encrypted;
    ...
    return encrypted;
}
```

这段代码仍不够完美，因为encrypted虽获定义但并没有任何实参作为初值。这意味着调用的是其default构造函数。许多时候你该对对象做的第一次事就是给他个值，通常是通过一个赋值动作达成，但是他比直接在构造时指定初值效率差，最佳做法：

```C++
std::string encryptPassword(const string& password) {
    ... // 如前，检查length
    std::string encrypted(password); // 通过copy构造函数定义并初始化
    ...
    return encrypted;
}
```

你不止应该延后变量的定义，直到非得使用该变量的前一刻为止，甚至应该尝试延后这份定义直到能够给他初值实参为止。如果这样，不仅能够避免构造非必要对象，还可以避免无意义的default构造行为。

但是在遇到循环的时候该怎么办？

```C++
// 方法A: 定义在循环外部：
Widget w;
for (int i = 0; i < n; ++i) {
    w = 取决于i的某个值;
    ...
}

// 方法B: 定义在循环的内部
for (int i = 0; i < n; ++i) {
    Widget w(取决于i的某个值);
    ...
}
```

两种方法的成本：
- 做法A: 1个构造函数 + 1个析构函数 + n个赋值操作
- 做法B: n个构造函数 + n个析构函数

如果class的一个赋值成本低于一组构造+析构的成本，方法A一般比较高效。尤其当n值很大的时候。否则做法B或许比较好。

> - 尽可能延后变量定义式的出现。这样做可以增加程序的清晰度并改善程序效率。

##### 27. 尽量少做转型动作

