---
title: Effective-C-改善程序与设计的55个具体做法(All)
date: 2020-1-7 00:02:37
tags:
- Effective C++
categories:
- C++
---

#### Effective-C-改善程序与设计的55个具体做法

1. 视C++为一个语言联邦
2. 尽量以const，enum，inline替换#define
3. 尽可能使用const
4. 确定对象被使用前已被初始化
5. 了解C++默默编写并调用哪些函数
6. 若不想使用编译器自动生成的函数，就该明确拒绝
7. 为多态基类使用virtual析构函数
8. 别让异常逃离析构函数
9. 绝不在构造函数和析构函数中调用virtual函数
10. 令operator= 返回一个reference to *this
11. 在operator= 中处理”自我赋值“（内容不太理解，看完后续章节再看）
12. 复制对象时勿忘记其每一个成分
13. 以对象管理资源
14. 在资源管理中小心copy行为
15. 在资源管理类中提供对原始资源的访问
16. 成对使用new和delete时要使用相同的形式
17. 以独立语句将newed对象置入智能指针
18. 让接口容易被正确使用，不易被误用
19. 设计class犹如设计type
20. 宁以pass-by-reference-to-const 替换pass-by-value
21. 必须返回对象时，别妄想返回其reference
22. 将成员变量声明为private
23. 宁以non-member、non-friend替换member函数
24. 若所有参数皆需类型转换，请为此采用non-member函数
25. 考虑写出一个不抛异常的swap函数
26. 尽可能延后变量定义式的出现时间
27. 尽量少做转型动作