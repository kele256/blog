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

C++ function接口，class接口，template接口...,每一种接口都是客户与你的代码互动的手段。理想上，如果客户企图使用某个接口而却没有获得他所预期的行为，这个代码不该通过编译；如果代码通过了编译，它的作为就该是客户所想要的。

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

默认情况下C++以pass by value的方式传递对象至函数。除非另外指定，否则函数参数都是以实参的复件为初值，而调用端所获得的也是函数返回值的一个复件。这些复件由对象的copy构造函数产出，这可能使pass-by-value成为耗时的操作。考虑以下class继承体系：

```C++
class Person {
public:
    Person();
    virtual ~Person();
    ...
private:
    std::string name;
    std::string address;
};

class Student : public Person {
public:
    Student();
    ~Student();
    ...
private:
    std::string schoolName;
    std::string schoolAddress;
};

bool validateStudent(Student s); // pass by value
Student plato;
bool platoIsOK = validateStudent(palto);
```

上述调用过程中参数的传递成本是”一次Student copy构造函数调用，再加上一次Student析构函数调用。再加上类内部的string，总体成本是六次构造函数和六次析构函数。

```C++
bool validateStudent(const Student& s); // 这种传递方式的效率要高得多
```

这种传递方式效率要高得多，没有任何构造函数或析构函数被调用，因为没有任何新对象被创建。

以引用方式传递参数也可以避免slicing（对象切割）问题。**当一个derived class对象以by value方式传递并被视为一个base class对象，base class的copy构造函数会被调用**，而造成此对象的行为像个derived class对象的那些特性化性质全被切割掉了，仅仅留下一个base class对象。

对内置类型而言，选择pass-by-value往往比引用传递效率高些，对于STL的迭代器和函数对象同样适用。但是迭代器和函数对象适用pass-by-value时需要确认它们不受切割问题的影响。

内置类型都相当小，但并不是所有小型types都适合pass-by-value。对象小并不意味其copy构造函数不昂贵。许多对象--包括大多数STL容器--内含的东西只比指针多一些，但是复制这种对象确需要承担“复制那些指针所指的每一样东西”。那将非常昂贵。

即使小型对象拥有并不昂贵的copy构造函数，还是可能有效率上的争议。某些编译器对待“内置类型”和“用户自定义类型”的态度截然不同，纵使两者拥有相同的底层表述。例如：某些编译器拒绝把只由一个double组成的对象放进缓存器内，却很乐意在一个正规基础上对光秃秃的doubles那么做。当这种事情发生时，就应该以引用方式传递此对象。

“小型用户自定义类型不是必然成为pass-by-value”的另一个理由是：作为一个用户自定义类型，其大小容易有所变化。一个type目前虽然小，将来也许会变大。甚至改用另一个C++编译器都可能改变type的大小。

一般而言，可以假设“值传递”的唯一对象就是内置类型和STL的迭代器和函数对象。

> - 尽量以pass-by-reference-to-const替换pass-by-value。前者通常比较高效，并且可以避免切割问题。
> - 以上规则并不适用于内置类型，以及STL的迭代器和函数对象。对他们而言，值传递往往比较适当。

##### 21. 必须返回对象时，别妄想返回其reference

一心一意根除值传递带来的种种坏处，坚定追求引用传递时，容易犯下一个致命错误：开始传递一些指向其实并不存在的对象。

考虑一个用以表现有理数的class，内含一个函数用来计算两个有理数的乘积：

```C++
class Rational {
public:
    Rational(int numerator = 0, int denominator = 1);
    ...
private:
    int n, d; // 分子和分母
    friend const Rational operator* (const Rational& lhs, const Rational& rhs); // 友元函数不是类的成员
};
```

这个版本的operator*使用by value的方式返回其计算结果。如果改用引用传递，就不需要copy构造和析构的成本。但是记住，**所谓引用只是个名称，代表某个既有对象。任何时候看到一个引用声明式，都应该立刻问自己，它的另外一个名称是什么？因为它一定是某物的另一个名称。**以上述operator*为例，如果它返回一个reference，后者一定指向某个既有的Rational对象，内含两个Rational对象的乘积。

我们当然不可能期望这样一个Rational对象在调用operator*之前就存在。也就是说，如果你有：

```C++
Rational a(1, 2); // a = 1/2
Rational b(3, 5); // b = 3/5
Ratioanl c = a * b; // c应该是3/10
```

期望原本就存在一个值为3/10的Rational对象并不合理。

函数创建新对象的途径有两个：在stack空间和heap空间。如果定义一个local变量，就是在stack空间创建对象。根据这个策略试写operator*如下：

```C++
const Rational& operator* (const Rational& lhs, const Rational& rhs) {
    Rational result(lhs.n * rhs.n, lhs.d * rhs.d); // 糟糕的代码！
    return result; // 返回一个local对象，而local对象在函数退出前就销毁了
}
```

避免上述的写法，因为目标是要避免调用构造函数，而result却必须由构造函数构造起来。更严重的是：这个函数返回的result引用，在函数退出前就已经被销毁了。

于是，考虑在heap内创建一个对象：

```C++
const Rational& operator* (const Rational& lhs, const Rational& rhs) {
    Rational *result = new Rational(lhs.n * rhs.n, lhs.d * rhs.d);
    return *result; // 更糟糕的写法
}
```

还是必须付出一个构造函数调用的代价，同时又有另一个问题：该谁对着被new出来的对象实施delete？

即使调用者谨慎，出于良好的意识，他们还是不能够在这样合情合理的用法下阻止内存泄露：

```C++
Rational w, x, y, z;
w = x * y * z;
```
这里同一个语句内调用了两次operator*，因而两次使用new， 也就需要两次delete。但是确没有合理的办法进行资源释放。因为没有合理的办法让他们取得operator* 返回的引用背后隐藏的那个指针。会造成内存泄露。

上述两种方法都产生了构造函数调用，有一种办法可以避免任何的构造函数调用：
```C++
const Rational& operator* (const Rational& lhs, const Rational& rhs) {
    static Rational result; // 又一堆烂代码
    result = ...;
    return result;
}
```

就像所有用上static对象的设计一样，这一个也立刻造成我们对多线程安全性的疑虑。更深层的瑕疵：

```C++
bool operator== (const Rational& lhs, const Rational& rhs);

Rational a, b, c, d;
...
if ((a * b) == (c * d)) {
    // 乘积相等时，做相应的动作
} else {
    // 乘积不等时，做相应的动作
}
```

if判断的结果总是为true，不论a、b、c的值是什么。代码的等价形式：

```C++
if (operator==(operator*(a, b), operator*(c, d)))
```

在operator==调用之前，已有两个operator*调用式起作用，每一个都返回reference指向operator*内部定义的static Rational对象。因此一直返回true。**（两次operator*调用的确各自改变了static Rational对象值，但由于他们返回的都是引用，因此调用端看到的永远是static Rational对象的“现值”）**。

综上所述：一个“必须返回新对象”的函数的正确写法是，就让那个函数返回一个新对象。
```C++
inline const Rational operator* (const Rational& lhs, const Rational& rhs) {
    return Rational(lhs.n * rhs.n, lhs.d * rhs.d);
}
```

当然需要承担operator*返回值的构造成本和析构成本，但是长远来看是为了获得正确行为而付出的一个小小代价。

> - 绝不要返回pointer或reference指向一个local stack对象，或返回reference指向一个heap-allocated对象，或返回pointer或referenc指向一个local static对象而有可能同时需要多个这样的对象。

##### 22.将成员变量声明为private

首先从语法一致性说起（条款18）。如果成员变量不是public，客户唯一能够访问对象的办法就是通过成员函数。如果public接口内的每样东西都是函数，客户就不需要在打算访问class成员时试着记住是否该使用小括号。

使用函数可以让你对成员变量的处理有更精确的控制。使用函数可以实现出“不准访问”，“只读访问”以及“读写访问”。如果通过函数访问成员变量，日后可改以某个计算替换这个成员变量，而class客户一点也不会知道class的内部实现已经起了变化。

将成员变量隐藏在函数接口的背后，可以为“所有可能的实现”提供弹性。

> - 切记将成员变量声明为private。这可以赋予客户访问数据的一致性，可细微划分访问控制、允诺约束条件获得保证，并提供class作者以充分的实现弹性。
> - protected并不比public更具封装性。从封装的角度来说，其实只有两种访问权限：private（提供封装）和其他（不提供封装）

##### 23. 宁以non-member、non-friend替换成员函数

加入有个class用来表示网页浏览器。这样的class可能提供的众多的函数中，有一些用来清除下载元素告诉缓存区、清除访问过的URLs的历史记录、以及移除系统中的所有cookies。

```C++
class WebBrowser {
public:
    ...
    void clearCache();
    void clearHistory();
    void removeCookies();
};
```

许多用户会想一整个执行所有这些动作，因此WebBrowser也提供这样一个函数：

```C++
class WebBrowser {
public:
    ...
    void clearEveryThing();
};
```

当然，这一机能也可由一个non-member函数调用适当的member函数而提供出来：

```C++
void clearBrowser(WebBrowser& wb) {
    wb.clearCache();
    wb.clearHistory();
    wb.removeCookies();
}
```

哪一个比较好呢？是clearEveryThing还是clearBrowser?

面向对象守则要求,数据以及操作数据的那些函数应该被捆绑在一块，这意味着他建议member函数是较好的选择。不幸的是这个建议不正确。这事基于面向对象真实意义的一个误解。面向对象守则要求数据应该尽可能被封装，然而与直观相反地，member函数clearEveryThing带来的封装性比non-member函数clearBrowser低。此外，提供non-member函数可允许对WebBrowser相关机能有较大的包裹弹性，而那最终导致较低的编译相依度，增加WebBrowser的可延伸性。因此在许多方面non-member做法比member做法好。

只因在意封装性而让函数称为class的non-member并不意味着它“不可以是另一个class的member"。例如我们可以令clearBrowser成为某个工具类的一个static member函数。只要它不是WebBrowser的一部分（或friend），就不会影响WebBrowser的private成员封装性。

一个像WebBrowser这样的class可能拥有大量便利函数，某些与书签有关，某些与打印有关，还有一些与cookie的管理有关...，通常客户只对其中的某些感兴趣。分离他们的最直接做法就是将书签相关便利函数声明于一个头文件，将cookie相关便利函数声明于另外一个头文件。。。这样允许客户只对他们所用的那一小部分系统形成编译相依。

> - 宁可拿non-member non-friend函数替换member函数。这样做可以增加封装性、包裹弹性和机能扩充性。

##### 24. 若所有参数皆需类型转换，请为此采用non-member函数

令class支持隐式类型转换通常是个糟糕的主意。当然这条规则有其例外，最常见的例外是在建立数值类型时。假设设计一个class用来表现有理数，允许整数隐式转换为有理数颇为合理。

```C++
class Rational {
public:
    Rational(int numberator = 0, int denominator = 1);
    int numberator() const;
    int denominator() const;
private:
    ...
};
```

你想支持算术运算诸如加法、乘法等等，但是是采用member函数? non-member函数？或可能的话由non-member friend函数？

首先member函数的写法：

```C++
class Rational {
public:
    ...
    const Rational operator* (const Rational& rhs) const;
};
```

这种写法使你能够将两个有理数相乘：

```C++
Rational oneEight(1, 8);
Rational oneHalf(1, 2);
Rational result = oneEight * oneHalf; // OK
result = result * oneEight; // OK
```

然而当进行混合式运算，会发现只有一半行得通：

```C++
result = oneHalf * 2; // OK
result = 2 * oneHalf; // ERROR
```

以函数形式重写上述两个式子：

```C++
result = oneHalf.operator*(2); // OK
result = 2.operator*(oneHalf); // ERROR!
```

oneHalf是一个内含operator* 函数的class的对象，所以编译器调用该函数，然而整数2并没有相应的class，也就没有operator* 成员函数。

注意上述调用成功的那个2，发生了隐式类型转换，编译器知道正在传递一个int，而函数需要的是Rational；但他知道只要调用Rational构造函数并赋予所提供的int，就可以变出一个适当的Rational来。如果构造函数指定explicit，则上述两个都将编译失败。

只有当参数被列于参数列内，这个参数才是隐式类型转换的合格参与者。

想要支持混合式算术运算，可行的方法是：让operator* 成为一个non-member函数。

```C++
const Rational operator*(const Rational& rhs, const Rational& lhs) {
    return Rational(lhs.numberator() * rhs.numberator(), lhs.denominator() * rhs.denominator());
}

Rational oneFourth(1, 4);
Rational result;
result = oneFourth * 2; // OK
result = 2 * oneFourth; // OK
```

不适用friend的原因是：无论何时如果可以避免friend函数就该避免。

> - 如果你需要为某个函数的所有参数（包括被this指针所指的那个隐喻参数）进行类型转换，那么这个函数必须是个non-member。

##### 25. 考虑写出一个不抛异常的swap函数

所谓swap两对象值，意思是将两对象的值彼此赋予对方。默认情况下swap动作可由标准程序库提供的swap算法完成。其典型的实现：

```C++
namespace std {
    template<typename T>
    void swap(T& a, T& b) { // 类型T支持copy(copy 构造函数和 copy assignment)
        T temp(a);
        a = b;
        b = temp;
    }
}
```

这个默认的swap实现版本十分平淡，涉及到了三个对象的复制。但是对某些类型而言，这些复制动作无一必要。

其中最主要的就是“以指针指向一个对象，内含真正数据”那种类型。这种设计的常见表现形式是所谓“pimpl手法”。如果以这种手法设计Widget class，看起来就会像这样：

```C++
class WidgetImpl {
public:
    ...
private:
    int a, b, c; // 可能有许多数据，意味着复制时间很长
    std::vector<double> v;
};

class Widget { // 这个class使用pimpl手法
public:
    Widget(const Widget& rhs);
    Widget& operator=(const Widget& rhs) {
        ...
        *pImpl = *(rhs.pImpl); // operator=的一般性实现细节，参照条款10，11，12
        ...
    }
    ...
private:
    WidgetImpl* pImpl; // 指针，所指对象内含Widget数据
};
```

一旦要置换两个Widget对象值，我们唯一需要做的就是置换其pImpl指针，但是默认的swap算法并不知道这一点。

我们希望能够告诉std::swap；当Widgets被置换时真正该做的是置换其内部的pImpl指针。下面是基本构想：

```C++
namespace std {
    template<>
    void swap<Widget> (Widget& a, Widget& b) {
        swap(a.pImpl, b.pImpl); // 目前还不能编译过
    }
}
```

这个函数一开始的“template<>"表示它是std::swap的一个全特化版本，函数名称之后的”\<Widget>"表示这一特化版本是针对“T是Widget"而设计。通常我们不能改变std命名空间内的任何东西，但可以为标准templates（如swap）制造特化版本，使它专属于我们自己的classes。以上作为正是如此。

```C++
class Widget { // 与前相同，唯一区别是增加swap函数
public:
    ...
    void swap(Widget& other) {
        using std::swap; // 这个声明之所以必要，稍后解释
        swap(pImpl, other.pImpl); // 若要置换Widgets就置换其pImpl指针
    }
    ...
};

namespace std {
    template<>
    void swap<Widget>(Widget& a, Widget& b) {
        a.swap(b); // 若要置换Widgets,调用其swap成员函数
    }
}
```

这种做法不仅能够通过编译，还与STL容器有一致性，因为所有STL容器也都提供有public swap成员函数和std::swap特化版本（用以调用前者）。

```C++
#include<iostream>
#include<vector>

using namespace std;

class WidgetImpl {
public:
    WidgetImpl()
    : a(0)
    , b(0)
    , c(0)
    {

    }

    WidgetImpl(int _a, int _b, int _c)
    : a(_a)
    , b(_b)
    , c(_c)
    {

    }

    WidgetImpl(WidgetImpl& rhs) {
        a = rhs.a;
        b = rhs.b;
        c = rhs.c;

        for (std::vector<double>::iterator iter = rhs.v.begin(); iter != rhs.v.end(); ++iter) {
            v.push_back(*iter);
        }
    }

    void addVec(double w) {
        v.push_back(w);
    }

    void printImpl() {
        cout << "a: " << a << endl;
        cout << "b: " << b << endl;
        cout << "c: " << c << endl;

        for (std::vector<double>::iterator iter = v.begin(); iter != v.end(); ++iter) {
            cout << *iter << endl; 
        }
    }

    WidgetImpl& operator=(WidgetImpl& rhs) {
        a = rhs.a;
        b = rhs.b;
        c = rhs.c;

        for (std::vector<double>::iterator iter = rhs.v.begin(); iter != rhs.v.end(); ++iter) {
            v.push_back(*iter);
        }
        return *this;
    }

    ~WidgetImpl() {

    }

private:
    int a, b, c;
    std::vector<double> v;
};

class Widget {
public:
    Widget(WidgetImpl* rhs)
    : pImpl(rhs)
    {

    }

    Widget(const Widget& rhs) {
        pImpl = rhs.pImpl;
    }

    void swap(Widget& other) {
        using std::swap;
        swap(pImpl, other.pImpl);
    }

    void printWidget() {
        pImpl->printImpl();
    }

    Widget& operator=(const Widget& rhs) {
        WidgetImpl* pOrig = pImpl;
        pImpl = new WidgetImpl(*rhs.pImpl);
        delete pOrig;
        return *this;
    }

private:
    WidgetImpl* pImpl;
};

namespace std {
    template<>
    void swap<Widget>(Widget& a, Widget& b) {
        a.swap(b);
    }
}

int main()
{
    WidgetImpl tmp(1, 2, 3);
    tmp.addVec(2.1);
    tmp.printImpl();

    WidgetImpl tmp1(4, 5, 6);
    tmp1.addVec(2.2);
    tmp1.printImpl();

    Widget tmp2(&tmp);
    tmp2.printWidget();

    Widget tmp3(&tmp1);
    tmp3.printWidget();

    swap(tmp2, tmp3);

    cout << "After swap" << endl;
    tmp2.printWidget();
    tmp3.printWidget();

    return 0;
}
```

然而假设Widget和WidgetImpl都是class template而非classes，也许可以试试将WidgetImpl内的数据类型加以参数化：

```C++
template<typename T>
class WidgetImpl {...};

template<typename T>
class Widget {...};
```

在Widget内放个swap成员函数跟以往一样。但是我们却在特化std::swap时遇上乱流：

```C++
namespace std {
    template<typename T>
    void swap< Widget<T> >(Widget<T>& a, Widget<T>& b) { // 错误，不合法！
        a.swap(b);
    }
}
```

我们企图偏特化（partially specialize)一个function template（std::swap)，但C++只允许对class templates偏特化，在function template身上偏特化是行不通的。

当打算偏特化一个function template时，惯常做法是简单地为它添加一个重载版本，像这样：

```C++
namespace std {
    template<typename T>
    void swap(Widget<T>& a, Widget<T>& b) { // std::swap的重载版本，也不合法
        a.swap(b);
    }
}
```

一般而言，重载function templates没有问题，但std是个特殊的命名空间，其管理规则也比较特殊。客户可以全特化std内的template，但是不可以添加新的templates（或classes或functions）到std里头。std的内容完全由C++标准委员会决定。

我们还是声明一个non-member swap让它调用member swap，但不再将那个non-member swap声明为std::swap的特化版本或重载版本。为求简化起见，假设Widget的所有相关机能都被置于命名空间WidgetStuff内，整个结果看起来便像这样：

```C++
namespace WidgetStuff {
    ...
    template<typename T>
    class Widget {...}; // 同前，内含swap成员函数

    ...
    template<typename T>
    void swap(Widget<T>& a, Widget<T>& b) { // WidgetStuff命名空间
        a.swap(b);
    }
}
```

这个做法对classes和class templates都行得通，所以似乎我们应该在任何时候都使用它。但是有一个理由使你应该为class特化std::swap，所以如果你想要你的”class 专属版“swap在尽可能多的语境下被调用，需要在该class所在命名空间内写一个non-member版本以及一个std::swap特化版本。

目前为止所写的每一样东西都和swap编写者有关。换位思考，从用户观点来看事情也有必要。假设你正在写一个function template，其内需要置换两个对象值：

```C++
template<typename T>
void doSomething(T& obj1, T& obj2) {
    ...
    swap(obj1, obj2);
    ...
}
```

应该调用哪个swap？下面是你希望发生的事：

```C++
template<typename T>
void doSomething(T& obj1, T& obj2) {
    using std::swap; // 令std::swap在此函数调用
    ...
    swap(obj1, obj2); // 为T型对象调用最佳swap版本
    ...
}
```

一旦编译器看到对swap函数的调用，它们便查找适当的swap并调用之。C++的名称查找法则确保找到global作用域或T所在之命名空间内的任何T专属的swap。如果T是Widget并位于命名空间WidgetStuff内，编译器会使用”实参取决之查找规则“，没有T专属之swap存在，编译器就使用std内的swap，这得感谢using声明式让std::swap在函数内曝光。然而即便如此编译器还是比较喜欢std::swap的T专属特化版，而非一般化的那个template。

因此，令适当的swap被调用是很容易的。需要小心的是，别为这一调用添加额外修饰符，因为那会影响C++挑选适当函数，假设你以如下方式调用swap：

```C++
std::swap(obj1, obj2); // 这是错误的swap调用方式
```

总结：如果swap默认实现版的效率不足，试着做以下事情：

1. 提供一个public swap成员函数，让它高效地置换你的类型的两个对象值。
2. 在你的class后template所在的命名空间内提供一个non-member swap，并令他调用上述swap成员函数。
3. 如果你正编写一个class（而非class template），为你的class特化std::swap。并令它调用你的swap成员函数。

> - 当std::swap对你的类型效率不高时，提供一个swap成员函数，并确定这个函数不抛出异常。
> - 如果你提供一个member swap，也该提供一个non-member swap用来调用前者。对于classes（而非templates),也请特化std::swap。
> - 调用swap时应针对std::swap使用using声明式，然后调用swap并且不带任何”命名空间资格修饰“。
> - 为”用户定义类型“进行std template全特化是好的，但千万不要尝试在std内加入某些对std而言全新的东西。