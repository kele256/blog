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

C++规则的设计目标之一是，保证”类型错误“绝不可能发生。不幸的是，转型破坏了类型系统。

首先回顾转型语法，因为通常有3中不同的形式，C风格的转型动作看起来像这样：

```C++
(T)expression // 将expression转型为T
// 函数风格的转型动作看起来像这样
T(expression)
```

这两种形式并无差别，称为”旧式转型“

C++还提供四种转型（常被称为new-style或C++style casts):

```C++
const_cast<T>(expression)
dynamic_cast<T>(expression)
reinterpret_cast<T>(expression)
static_cast<T>(expression)
```

各有不同的目的：

- const_cast 通常是用来将对象的常量性转除，它也是唯一有此能力的C++ style转型操作符
- dynamic_cast 主要用来执行”安全向下转型“，也就是用来决定某个对象是否归属继承体系中的某个类型。它是一个唯一一个无法由旧式语法执行的动作，也是唯一可能耗费重大运行成本的转型动作。
- reinterpret_cast 意图执行低级转型，实际动作（及结果）可能取决于编译器，这也就表示它不可移植。例如将一个pointer to int转型为一个int。这一类转型在低级代码以外很少见，是在讨论原始内存写出一个调试用的分配器时，见条款50.
- static_cast 用来强迫隐式转换，例如将non-const对象转为const对象，或将int转为double等等。它也可以用来执行上述多种转换的反向转换，例如将void* 指针转为typed指针，将pointer-to-base转为pointer-to-derived.但无法将const转为non-const。

许多程序员相信。转型其实什么都没做，只是告诉编译器把某种类型视为另一种类型。这是错误得观念。任何一种类型转换（不论是通过转型操作而进行的显示转换，或通过编译器完成的隐式转换）往往真的令编译器编译出运行期间执行的码，例如在这段程序中：

```C++
int x, y;
...
double d = static_cast<double>(x)/y; // x除以y,使用浮点数除法
```

将int x转换为double几乎肯定会产生一些代码，因为在大部分计算器体系结构中，int的底层表述不同于double的底层表述。

```C++
class Base { ... };
class Derived : public Base { ... };
Derived d;
Base* pb = &d;
```

这里我们不过是建立一个base class指针指向一个derived class对象，但有时候上述的两个指针值并不相同。这种情况下会有个偏移量在运行期间被施行于Derived* 指针身上，用以取得正确的Base* 指针值。

上个例子表明，单一对象（例如一个类型为Derived的对象）可能拥有一个以上的地址（例如”以Base* 指向它“时的地址和“ 以Derived*指向它”时的地址。C不可能发生这种事，java不可能发生这种事，C#也不可能发生这种事。但是C++能！实际上一旦使用多重继承，这事几乎一直发生着。即使在单一继承中也可能发生。

请注意，说的是有时候需要一个偏移量。对象的布局方式和它们的地址计算方式随编译器的不同而不同，那意味着“由于知道对象如何布局”而设计的转型，在某一平台行得通，在其他平台并不一定行得通。

我们很容易写出似是而非的代码（在其他语言中也许真是对的）：

```C++
class Window {
public:
    virtual void onResize() { ... }
};

class SpecialWindow : public Window {
public:
    virtual void onResize() {
        static_cast<Window>(*this).onResize();
        ... // 这里进行SpecialWindow专属行为
    }
};
```

这段程序将*this转型为Window，对函数onResize的调用也因此调用了Window::onResize. 但恐怕你没有想到，它调用的并不是当前对象身上的函数，而是稍早转型动作所建立的一个” *this对象之base class成分”的暂时副本身上的onResize! 如果Window::onResize修改了对象内容，当前对象其实没有被改动，改动的是副本。

解决方法是，去掉转型动作：

```C++
class SpecialWindow : public Window {
public:
    virtual void onResize() {
        Window::onResize();
        ...
    }
};
```

之所以需要dynamic_cast，通常是因为想在一个你认定为derived class对象身上执行derived class操作函数，但你的手上却只有一个“指向Base”的pointer或reference，你只能靠他们来处理对象。有两个一般性做法可以避免这个问题。

第一，使用容器并在其中存储直接指向derived class对象的指针（通常是智能指针），如此便消除了“通过base class接口处理对象”的需要。假设先前的Window/SpecialWindow继承体系中只有SpecialWindow才支持闪烁系统，试着不要这么做：

```C++
class Window { ... };
class SpecialWindow : public Window {
public:
    void blink();
    ...
};

typedef std::vector<std::tr1::shared_ptr<Window>> VPW;
VPW winPtrs;
...
for (VPW::iterator iter = winPtrs.begin(); iter != winPtrs.end(); ++iter) {
    if (SpecialWindow* psw = dynamic_cast<SpecialWindow*>(iter->get())) { // 不希望使用dynamic_cast
        psw->blink();
    }
}
```

应该这样做：

```C++
typedef std::vector<std::tr1::shared_ptr<SpecialWindow>> VPSW;
VPSW winPtrs;
...
for (VPSW::iterator iter = winPtrs.begin(); iter != winPtrs.end(); ++iter) {
    (*iter)->blink();
}
```

这种做法无法在同一个容器内存储指针“指向所有可能之各种Window派生类”。如果真要处理多种窗口类型，可能需要多个容器。

另一种做法可让你通过base class接口处理“所有可能之各种Window派生类”，那就是在base class中提供virtual函数，做你想对各个Window派生类做的事。举个例子，虽然只有SpecialWindow可以闪烁，但或许将闪烁函数声明于base class内并提供“什么也不做”的默认实现代码是有意义的：

```C++
class Window {
public:
    virtual void blink() { } // 默认实现代码什么也没做
    // 条款34会说明默认实现代码可能是个馊主意
    ...
};
class SpecialWindow : public Window {
public:
    virtual void blink() {...};
};

typedef std::vector<std::tr1::shared_ptr<Window>> VPW;
VPW winPtrs;
...
for (VPW::iterator iter = winPtrs.begin(); iter != winPtrs.end(); ++iter) {
    (*iter)->blink();
}
```

绝对必须避免的一件事是所谓的“连串dynamic_cast"，也就是看起来像这样的东西：

```C++
class Window {...};
...
typedef std::vector<std::tr1::shared_ptr<Window>> VPW;
VPW winPtrs;
...
for (VPW::iterator iter = winPtrs.beigin(); iter != winPtrs.end(); ++iter) {
    if (SpecialWidow *psw1 = dynamic_cast<SpecialWindow1>(iter->get())) {
        ...
    }
    else if (SpecialWindow *psw2 = dynamic_cast<SpecialWindow2>(iter->get()) {
        ...
    })
    else if ...
}
```

这样的代码又大又慢，而且基础不稳，每次Window class继承体系一有改变，所有这一类代码都必须再次检阅看看是否需要修改。

> - 如果可以，尽量避免转型，特别是在注重效率的代码中避免dynamic_casts.如果有个设计需要转型动作，试着发展无需转型的替代设计
> - 如果转型是有必要的，试着将他隐藏于某个函数背后。客户随后可以调用该函数，而不需将转型放进他们自己的代码中。
> - 宁可使用C++ style转型，不要使用旧式转型。前者很容易辨识出来，而且也比较由着分门别类的职责。

##### 28. 避免返回handles指向对象内部成分

