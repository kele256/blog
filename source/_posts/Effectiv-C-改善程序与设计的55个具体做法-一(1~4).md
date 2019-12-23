---
title: Effectiv C++ 改善程序与设计的55个具体做法(1~4)(一)
date: 2019-11-26 00:49:41
tags:
- Effective C++
categories:
- C++
---

#### 一、让自己习惯C++

##### 1、 视C++为一个语言联邦

可以将C++视为由C、Object-Oriented C++、Template C++、STL等次语言组成的联邦，当从一个次语言移往另一个次语言时，守则可能改变。
<!-- more -->

例如对内置（也就是C-like）类型而言pass-by-value通常比pass-by-reference高效，但当从C Part of C++移往Object-Oriented C++,由于用户自定义构造函数和析构函数的存在，pass-by-reference-const往往更好。运用Template C++时尤其如此，然而一旦跨入STL，迭代器和对象都是在C指针之上塑造出来的，所以对STL的迭代器和函数对象而言，pass-by-value通常更高效。

##### 2、尽量以const，enum，inline替换 #define

当我们以常量替换#define，有两种情况值得注意：

- 定义常量指针
由于常量定义式通常放在头文件内，因此有必要将指针（而不是指针所指之物）声明为const。
- class专属常量
为了将常量的作用域限制于class内，必须让他成为class的一个成员：而为确保此常量至多只有一份实体，必须让它成为一个static成员。

```C++
// 位于头文件
class GamePlayer {
private:
    static const int NumTurns = 5; // 常量声明式
    int scores[NumTurns]; // 使用该常量
    ...
}
```

注意所看到的NumTurns是声明式而非定义式。通常C++要求对所使用的任何东西提供一个定义式，但如果它是个class专属常量又是static且为整数类型（ints, chars, bools），则需特殊处理。**只要不取他们的地址**，你可以声明并使用而无需提供定义式。但如果要取它的地址，就必须提供另外的定义式如下：

```C++
// 位于实现文件内
const int GamePlayer::NumTurns; // NumTurns的定义
```

由于class常量在声明式已经获得初值，因此定义时不可以在设定初值。

当在class编译期间需要一个class常量值，例如在上述GamePlayer::scores的数组声明式内（老的版本编译器可能不允许“static 整数型常量在class内完成初值设定”）。可以改用“The enum hack"补偿做法。如下：

```C++
class GamePlayer {
private:
    enum {NumTurns = 5};
    int scores[NumTurns];
}
```

enum hack的行为在某些方面比较像”#define“而不像const。例如**取一个const的地址是合法的，但取一个enum的地址就不合法**，而取一个#define的地址通常也不合法。如果不像让别人获得一个pointer或reference指向你的某个整数常量，enum可以帮助实现这个约束。

> - 对于单纯常量，最好以const对象或enums替换#define。
> - 对于形似函数的宏，最好改用inline 函数替换#defines。

##### 3、尽可能使用const

- const可以指定一个语义约束，编译器会帮助确保这条约束不被违反。

```C++
char greeting[] = "Hello";
char* p = greeting; // 非const指针，非const数据
const char* p = greeting; //非const指针，const数据
char* const p = greeting; //const指针，非const数据
const char* const p = greeting; //const指针，const数据
char const *p = greeting; // 非const指针，const数据，同2
```

- STL迭代器也可以声明为const:

```C++
std::vector<int> vec;
...
const std::vector<int>::iterator iter = 
    vec.begin(); // iter就相当于一个指针，const 指针表示iter不能指向不同的东西，但是所指东西的值是可以改变的。
*iter = 10; // 没问题
++iter; // 错误

std::vector<int>::const_iterator cIter =
    vec.begin(); // const_iterator的作用像个const T*,指针可以改变，但是所指向的值不能改变。
*cIter = 10; // 错误
++cIter; // 正确
```

- 令函数返回一个常量，往往可以降低客户错误而造成的意外。考虑如下:

```C++
class Rational {...};
const Rational operator* (const Rational& lhs, const Rational& rhs);

Rational a, b, c;
...
(a * b) = c; // 在a * b的结果上调用operator*
```

- const 成员函数

将const实施于成员函数的重要性基于以下两个理由：

1、 可以使class接口比较容易被理解。(很容易看出哪个接口可以改动对象内容)

2、 他们使操作”const“对象成为可能，改善C++程序效率的一个根本办法是以pass by reference-to-const方式传递对象。
> 两个成员函数如果只是常量性不同，可以被重载。

```C++
#include<iostream>
#include<string>
using namespace std;

class TextBlock {
public:
    TextBlock(char* chs) {
        text = chs;
    }

    const char& operator[](size_t position) const
    {
        cout << "const" << endl;
        return text[position];
    }

    char& operator[](size_t position) {
        cout << "non-const" << endl;
        return text[position];
    }

    void printTextBlock() {
        cout << text << endl;
    }

private:
    std::string text;
};

int main()
{
    char* test = "Hello";
    TextBlock tb(test); // non-const

    cout << tb[1] << endl;
    tb[1] = 'a';
    cout << tb[1] << endl;
    tb.printTextBlock();

    const TextBlock ctb(test);
    cout << ctb[2] << endl; // const
    return 0;
}
```

**const 成员函数不可以更改对象内任何non-static成员变量(成员变量中含有指针，指针所指向的值可以改变)。**
**使用mutable关键字可以释放掉non-static成员变量的const约束(bitwise constness)**

mutable关键字修饰的非静态成员变量可能总是会被修改,即使在const成员函数内部。

- 在const和non-const成员函数中避免重复

```C++
class TextBlock {
public:
    ...
    const char& operator[](std::size_t position) const
    {
        ... // 边界检查
        ... // 日志数据访问
        ... // 检验数据完整性
        return text[position];
    }

    char& operator[](std::size_t position)
    {
        ... // 边界检查
        ... // 日志数据访问
        ... // 检验数据完整性
        return text[position];
    }

private:
    std::string text;
};
```

上述代码发生代码重复以及伴随的编译时间，维护代码膨胀等问题。即使将边界检查/日志数据访问/检查数据完整性等转移到另一个成员函数，还是会重复一些代码
例如函数调用，两次return等。

真正该做的是实现operator[]的机能一次并使用他两次。

```C++
class TextBlock {
public:
    ...
    const char& operator[](std::size_t position) const
    {
        ...
        ...
        ...
        return text[position];
    }

    char& operator[](std::size_t position) // nont-const内部调用const op[]
    {
        return const_cast<char&>(static_cast<const TextBlock&>(*this)[position]);
        // 将op[]返回值的const转除，为*this加上const，调用const op[]
    }
    ...
};
```

> - 将某些东西声明为const可帮助编译器侦测出错误用法。const可被施加于任何作用域内的对象，函数参数，函数返回类型，成员函数本体。
> - 编译器强制实施bitwise constness,但是编写程序时应该使用"概念上的常量性"
> - 当const和non-const成员函数有着实质等价的实现时，令non-const版本调用const版本可以避免代码重复。

##### 4. 确定对象被使用前已先被初始化

这个规则很容易奉行，但是容易混淆赋值和初始化：

```C++
class PhoneNumber {...};
class ABEntry {
public:
    ABEntry(const std::string& name, const std::string& address, const std::list<PhoneNumber>& phones);

private:
    std::string theName;
    std::string theAddress;
    std::list<PhoneNumber> thePhones;
    int numTimesConsulted;
};

ABEntry::ABEntry(const std::string& name, const std::string& address, const std::list<PhoneNumber>& phones)
{
    theName = name;  // 这些都是赋值而非初始化
    theAddress = address;
    thePhones = phones;
    numTimesConsulted = 0;
}
```

赋值操作会使ABEntry对象带有期望的值，但不是最佳做法。C++规定，**对象成员的初始化动作发生在进入构造函数本体之前**。在ABEntry构造函数内，theName等都不是被初始化，而是被赋值。初始化发生的时间更早，发生于这些成员的default构造函数被自动调用时(比进入ABEntry构造函数本体的时间更早)。但是这对numTimesConsulted不为真，因为他属于内置类型，不保证一定在你所看到的那个赋值动作的时间点之前获得初值。

较佳的写法是：

```C++
ABEntry::ABEntry(const std::string& name, const std::string& address, const std::list<PhoneNumber>& phones)
: theName(name)
, theAddress(address)
, thePhones(phones)
, numTimesConsulted(0) // 这些都是初始化
{

}
```

这个构造函数和上一个的最终结果相同，但通常效率较高。基于赋值的版本首先调用default构造函数为theName/theAddress/thePhones设初值，然后再对他们赋予新值。

对于内置类型numTimeConsulted其初始化和赋值的成本相同。

有些情况下即使面对的成员变量属于内置类型，也一定要使用初始化列表。如果成员变量是const或references，他们就一定需要初值，不能被赋值。

C++有着十分固定的“成员初始化次序”：base classes更早于其derived classes被初始化，而class成员变量总是以其声明次序被初始化。

> - 为内置类型对象进行手动初始化，因为C++不保证初始化他们
> - 构造函数最好使用初始化列表， 而不要在构造函数本体内使用赋值操作。
> - 为消除“跨编译单元初始化次序问题”，以local static对象替换non-local static 对象
