---
title: Effective-C-改善程序与设计的55个具体做法(13~17)(三)
date: 2019-12-17 20:42:11
tags:
- Effective C++
categories:
- C++
---

#### 三、资源管理

所谓资源就是，一旦用了它，将来必须还给系统。C++中最常使用的资源就是动态分配内存，其他常见的资源还包括文件描述器(file descriptors)，互斥锁，图形界面中的字型和笔刷，数据库连接，以及网络sockets,不论哪一种资源，当不再使用它时，必须将它归还给系统。

##### 13. 以对象管理资源

假设我们使用一个用来模拟投资行为（例如股票，债券等等）的程序库，其中各式各样的投资类型继承自一个root clas Investment;

```C++
class Investment { ... }; // "投资类型"继承体系中的root class

// 通过工厂函数供应我们某特定的Investment对象
Investment* createInvestment();

void f()
{
    Investment* pInv = createInvestment(); // 调用factory函数
    ...
    delete pInv;
}
```

“...”区域可能会含有return语句，这样就不会执行delete动作。

为确保createInvestment返回的资源总是被释放，我们需要将资源放进对象内，当控制流离开f，该队象的析构函数会自动释放那些资源。

许多资源被动态分配于heap内而后被用于单一区块或函数内。它们应该在控制流离开那个区块或函数时被释放。标准程序库提供的auto-ptr正是针对这种形势而设计的特制的产品。auto-ptr是个类指针对象，也就是所谓的智能指针。下面是如何使用：

```C++
void f()
{
    std::auto_ptr<Investment> pInv(createInvestment());
    ... // 就像以前使用pInv，不必delete，经由auto_ptr的析构函数自动删除pInv
}
```

上述简单的例子示范”以对象管理资源“两个关键想法:

- 获得资源后立刻放进管理对象内
- 管理对象利用析构函数确保资源被释放

为了防止多个auto_ptr同时指向同一对象，auto_ptr有一个不寻常的性质：**若通过copy构造函数或copy assignment操作符复制他们，它们会变成null，而复制所得的指针将取得资源的唯一使用权。**

这一诡异的复制行为，意味着auto_ptr并非管理动态资源的好方法，例如：STL容器要求其元素发挥”正常的“复制行为，因此这些容器容不得auto_ptr。

auto_ptr的替代方案是”引用计数型智能指针“，它可以持续追踪共有多少对象指向某笔资源，并且在无人指向它时自动删除该资源。引用计数型智能指针无法打破环状引用（两个其实已经没被使用的对象彼此互指，因而好像还处在”被使用“状态）。

```C++
void f()
{
    ...
    std::tr1::shared_ptr<Investment> pInv(createInvestment());
    ...
}

void f()
{
    ...
    std::tr1::shared_ptr<Investment> pInv1(createInvestment());
    std::tr1::shared_ptr<Investment> pInv2(pInv1);
    pInv1 = pInv2;
    ...
}
```

auto_ptr和tr1::shared_ptr两者都在其析构函数内做delete而不是delete[]动作。意味着不能用在动态分配而得的array身上。

##### 14. 在资源管理中小心copy行为

自己建立资源管理类，假设我们使用C API函数处理类型为Mutex的互斥器对象共有lock和unlock两个函数可以使用：

```C++
void lock(Mutex* pm); // 锁定pm所指的互斥器
void unlock(Mutex* pm) // 将互斥器解除锁定

class Lock {
public:
    explicit Lock(Mutex* pm)
    : mutexPtr(pm)
    {
        lock(mutexPtr);
    }

    ~Lock() {
        unlock(mutexPtr);
    }
private:
    Mutex* mutexPtr;
};

Mutex m; // 定义所需的互斥器
...
{ // 建立一个区块用来定义critical section
    Lock m1(&m); // 锁定互斥器
    ... // 执行critial section内的操作
} // 在区块最末尾，自动解除互斥器锁定
```

如果某个Lock对象被复制，会发生什么？

```C++
Lock m11(&m); // 锁定m
Lock m12(m11); // 将m11复制到m12
```

大多数情况会选择以下两种可能：

- 禁止复制

许多时候允许RAII对象被复制不合理。如果复制动作对RAII class不合理，便应该禁止复制，见条款6

- 对底层资源祭出”引用计数法“

有时候我们希望保有资源，直到它的最后一个使用者（某对象）被销毁。这种情况下复制RAII 对象时，应该将资源的”被引用数“递增。tr1::shared_ptr便是如此.

通常只要内含一个tr1::shared_ptr成员变量，RAII classes便可实现reference-counting copying行为。如果前述的Lock打算使用引用计数，它可以改变mutexPtr的类型，将它从Mutex*改为tr1::shared_ptr\<Mutex\>。然而很不幸tr1::shared_ptr的缺省行为是“当引用次数为0时删除其所指物”，那不是我们所要的行为。当我们用上一个Mutex，我们想要做的释放动作是解除锁定而非删除。

幸运的是tr1::shared_ptr允许指定所谓的“删除器”，那是一个函数或函数对象，当引用次数为0时便被调用（此机能并不存在于auto_ptr-他总是将其指针删除）。删除器对tr1::shared_ptr构造函数而言是可有可无的第二参数，所以代码看起来像这样：

```C++
class Lock {
public:
    explicit Lock(Mutex* pm) // 以某个Mutex初始化shared_ptr
    : mutexPtr(pm, unlock) // 并以unlock函数为删除器
    {
        lock(mutexPtr.get()); // 条款15谈到“get"
    }
private:
    std::tr1::shared_ptr<Mutex> mutexPtr; // 使用shared_ptr替换raw pointer
};
```

本例的Lock class不再声明析构函数。因为没有必要。条款5说过class析构函数（无论是编译器生成的，或用户自定的）会自动调用其non-static成员变量（本例为mutexPtr)的析构函数。而mutexPtr的析构函数会在互斥器的引用次数为0时自动调用删除器（unlock）。

- 复制底部资源

对一份资源可以拥有任意数量的复件。而需要”资源管理类"的唯一理由是，当你不再需要某个复件时确保它被释放。在此情况下复制资源管理对象，应该同时也复制其所包裹的资源（深度拷贝）。

- 转移底部资源的拥有权

极少数的情况下可能希望确保永远只有一个RAII对象指向一个未加工资源（raw resource），即使RAII对象被复制依然如此。此时资源的拥有权会从被复制物转移到目标物。如条款13所述（auto_ptr)。

> - 复制RAII对象必须一并复制它所管理的资源，所以资源的copying行为决定RAII对象的copying行为。
> - 普遍而常见的RAII class copy行为是：抑制copying，实行引用计数法。不过其他行为也有可能被实现。

##### 15. 在资源管理类中提供对原始资源的访问

完美的情况是依赖资源管理类来处理所有和资源之间的互动，而不是直接处理原始资源，但是很多接口都是直接处理原始资源，所以需要提供对原始资源的访问。

有两个方法可以达成目标：显示转换和隐式转换。

tr1::shared_ptr和auto_ptr都提供一个get成员函数，用来执行显式转换，也就是它会返回智能指针内部的原始指针（的复件）。

就像（几乎）所有智能指针一样，tr1::shared_ptr和auto_ptr也重载了指针取值操作符（operator->和operator*)，它们允许隐式转换至底部原始指针。

```C++
class Investment { // investment继承体系的根类
public:
    bool isTaxFree() const;
    ...
};

Investment* createInvestment(); // factory函数
std::tr1::shared_ptr<Investment> pil(createInvestment());

bool taxable1 = !(pil->isTaxFree()); // 经由operator->访问资源
...
std::auto_ptr<Investment> pi2(createInvestment());

bool taxable2 = !((*pi2).isTaxFree()); // 经由operator*访问资源
```

由于有时候还是必须取得RAII对象内的原始资源，某些RAII class 设计者提供一个隐式转换函数。考虑下面这个用于字体的RAII class：

```C++
FontHandle getFont(); // C API
void releaseFont(FontHandle fh); // 来自同一组C API

class Font { // RAII class
public:
    explicit Font(FontHandle fh) // 获得资源
    : f(fh)
    { }

    ~Font() { releaseFont(f); } // 释放资源
    FontHandle get() const { return f; } // 显示转换函数
    operator FontHandle() const { return f; } // 隐式转换函数
private:
    FontHandle f; // 原始（raw)字体资源
};
```

> - APIs往往要求访问原始资源，所以每一个RAII class应该提供一个“取得其所管理的资源”的方法。
> - 对原始资源的访问可能经由显式转换或隐式转换。一般而言显式转换比较安全，但隐式转换对用户比较方便。

##### 16. 成对使用new 和delete时要采取相同的形式

数组的delete需要使用delete[]。因为单一对象的内存布局一般而言不同于数组的内存布局。更明确的说，数组所用的内存通常还包括“数组大小”的记录，以便delete知道需要调用多少次析构函数。单一对象的内存则没有这笔记录。

这个规则对于喜欢用typedef的人也很重要，因为它意味着typedef的作者必须说清楚，当程序员以new创建该种类型对象时，该以哪一种delete形式删除之。考虑下面这个typedef：

```C++
typedef std::string AddressLines[4] //每个人的地址有4行，每行是一个string
std::string pa1 = new AddressLines; // 注意，“new AddressLines"返回一个string*
// 就像new string[4]一样
// 就必须匹配”数组形式“的delete：
delete pa1; // 错误
delete[] pa1 // 正确
```

> - 如果在new表达式中使用[], 必须在相应的delete表达式中也使用[]。如果在new表达式中不使用[],一定不要在相应的delete表达式中使用[]。

##### 17. 以独立语句将newed对象置入智能指针

假设我们有个函数用来揭示处理程序的优先权，另一个函数用来在某动态分配所得的Widget上进行某些带有优先权的处理：

```C++
int priority();
void processWidget(std::tr1::shared_ptr<Widget> pw, int priority);

// 调用
processWidget(std::tr1::shared_ptr<Widget>(new Widget), priority());
```

在调用processWidget之前，编译器必须创建代码，做以下三件事：

- 调用priority
- 执行”new Widget"
- 调用tr1::shared_ptr构造函数

C++编译器以什么顺序来完成这些事情？首先可以确定“new Widget"一定执行于tr1::shared_ptr构造函数之前，但是对priority的调用则可以排在第一或第二或第三执行。如果编译器选择以第二的顺序执行它，一旦priority的调用导致异常，在此情况下”new Widget“返回的指针将会遗失，因为它尚未被置入tr1::shared_ptr内。因为在”资源被创建“和”资源被转换为资源管理对象“两个时间点之间有可能发生异常干扰，从而导致内存泄漏。

避免这类问题的方法：使用分离语句，分别写出（1）创建Widget. (2) 将它置入一个智能指针内，然后再把那个智能指针传给processWidget:

```C++
std::tr1::shared_ptr<Widget> pw(new Widget) // 在单独语句内以智能指针存储newed所得对象
processWidget(pw, priority()); // 这个动作不会造成内存泄漏
```

> - 以独立语句将newed对象存储于(置入)智能指针内。如果不这样做，一旦异常被抛出，有可能导致难以察觉的内存泄漏
