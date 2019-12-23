---
title: Effective-C-改善程序与设计的55个具体做法(四)(18~25)
date: 2019-12-17 20:42:37
tags:
- Effective C++
categories:
- C++
---

#### 四. 设计与声明

##### 18. 让接口容易被正确使用，不易被误用

C++在接口之海漂浮。function接口，class接口，template接口...,每一种接口都是客户与你的代码互动的手段。理想上，如果客户企图使用某个接口而却没有获得他所预期的行为，这个代码不该通过编译；如果代码通过了编译，它的作为就该是客户所想要的。

```C++
class Date {
public:
    Date(int month, int day, int year);
    ...
};
```

用户在使用上述接口时，很容易犯下至少两个错误：第一，错误的次序传递参数。第二，传递无效的月份或参数。

上述易产生的错误可以通过导入新类型来预防。分别导入外覆类型来区别天数，月份和年份

```C++
struct Day {
    explicit Day(int d)
    : val(d) {
        int val;
    };
    ...
    ...
};

class Date {
public:
    Date(const Month& m, const Day& d, const Year& y);
}
```

限制自定义类型的值：

```C++
class Month {
public:
    static Month Jan() { return Month(1); }
    static Month Feb() { return Month(2); }
    ...
private:
    explicit Month(int m);
};

Date d(Month::mar(), Day(30), Year(1995));
```

为了提供行为一致的接口，避免无端与内置类型不兼容"除非有好理由，否则应该尽量令你的types的行为与内置types一致"。

任何接口如果要求客户必须记得做某些事情，就是有着”不正确使用“的倾向。

> - 好的接口很容易被正确使用，不容易被误用。应该在所有的接口中努力达成这些性质
> - "促进正确使用”的办法包括接口的一致性，以及与内置类型的行为兼容
> - “阻止误用”的办法包括建立新类型，限制类型上的操作，束缚对象值，以及消除客户的资源管理责任
> - tr1::shared_ptr支持定制型删除器。这可防范DLL问题，可被用来自动解除互斥锁等等。

##### 19. 设计class犹如设计type

设计优秀的classes是一项艰巨的工作，几乎每一个class都要求你面对以下提问：

- 新type的对象应该如何被创建和销毁？

这会影响到你的class的构造函数和析构函数以及内存分配函数和释放函数的设计

- 对象的初始化和对象的赋值该有什么样的差别？

这个答案决定你的构造函数和赋值操作符的行为，以及其间的差异。很重要的是别混淆了“初始化”和“赋值”，因为他们对应于不同的函数调用。

- 新type的对象如果被passed by value，意味着什么？

记住，copy构造函数用来定义一个type的pass-by-value该如何实现

- 什么是新type的“合法值”？

对class的成员变量而言，通常只有某些数值集是有效的。那些数值集决定了你的class必须维护的约束条件，也就决定了你的成员函数必须进行的错误检查工作。

- 你的新type需要配合某个继承图系吗？

如果你继承自某些既有的classes，你就受到哪些classes的设计的束缚，特别是受到“它们的函数是virtual或non-virtual”的影响。如果你允许其他calsses继承你的class，那会影响你所声明的函数-尤其是析狗函数--是否为virtual。

- 你的新type需要什么样的转换？

你的type生存与其他很多types之间，因而彼此该有转换行为吗？如果你希望允许类型T1之物被隐士转换为类型T2之物，就必须在class T1内写一个类型转换函数（operator T2）或在class T2内写一个non-explicit-one-argument（可被单一实参调用）的构造函数。如果你只允许explicit构造函数存在，就得专门写出负责执行转换的函数，且不得为类型转换操作符或non-explicit-one-argument构造函数（条款15有隐式和显式转换函数的范例）

- 什么样的操作符和函数对此新type而言是合理的？

这个问题的答案决定将为你的class声明哪些函数。其中某些该是member函数，某些则否

- 什么样的标准函数应该驳回？

那些正是你必须声明为private者

- 谁该取用新type的成员？

这个提问可以帮你决定哪个成员为public，哪个为protected，哪个为private。它也帮助你决定哪一个classes或functions应该是friends，以及将他们嵌套于另一个之内是否合理。

- 什么是新type的”未声明“接口？

它对效率、异常安全性（见条款29）以及资源运用（例如多任务锁定和动态内存）提供何种保证？你在这些方面提供的保证将为你的class实现代码加上相应的约束条件。

- 你的新type有多么一般化？

或许你其实并非定义一个新type，而是定义一整个types家族，果真如此你就不该定义一个新class，而是应该定义一个新的class template。

- 你真的需要一个新type吗？

如果只是定义新的derived class以便为既有的class添加机能，那么说不定单纯定义一或多个non-member函数或templates，能够达到目标。

> - Class的设计就是type的设计。在定义一个新的type之前，请确定你已经考虑过本条款的所有讨论主题。

##### 20. 宁以pass-by-reference-to-const 替换pass-by-value


