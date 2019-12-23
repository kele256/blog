---
title: Effective C++改善程序与设计的55个具体做法(5~12)(二)
date: 2019-12-05 23:29:54
tags:
- Effective C++
categories:
- C++
---

#### 二、构造/析构/赋值运算

##### 5. 了解C++默默编写并调用哪些函数
<!-- more -->
如果自己没有声明，编译器会声明一个copy构造函数，一个copy assignment操作符和一个析构函数。此外，如果没有声明任何构造函数，编译器也会声明一个default构造函数，所有这些函数都是pulic且inline

```C++
class Empty{};

class Empty {
public:
    Empty() {...}  // default 构造函数
    Empty(const Empty& rhs) {...}  // copy构造函数
    ~Empty() {...}  // 析构函数(编译器产生的析构是non-virtual，除非这个class的base class自身声明中有virtual析构)

    Empty& operator=(const Empty& rhs) {...} // copy assignment操作符
};
```

**只有当这些函数被需要（被调用），它们才会被编译器创建出来。**

对于copy构造函数和copy assignment操作符，编译器创建的版本只是单纯的将来源对象的每一个non-static成员变量拷贝到目标对象。

```C++
template<typename T>
class NamedObject {
public:
    NamedObject(const char* name, const T& value);
    NamedObject(const std::string &name, const T& value);
    ...
private:
    std::string nameValue;
    T objectValue;
};
```

由于其中声明了一个构造函数，编译器于是不再为它创建默认构造函数。

```C++
#include<iostream>
using namespace std;

template<typename T>
class NamedObject {
public:
    NamedObject(std::string& name, const T& value) 
    : objectValue(value)
    , nameValue(name)
    {

    }

public:
    std::string& nameValue;
    const T objectValue;
};

int main()
{
    std::string newDog("Persephone");
    std::string oldDog("Satch");

    NamedObject<int> p(newDog, 2);
    NamedObject<int> s(oldDog, 36);
    p = s; // 错误
    return 0;
}
```

C++不允许让“reference改指向不同对象”。对于这种情况，C++的响应是拒绝编译那一行赋值动作。面对内含“const”成员的class，编译器的反应也一样。必须自己定义copy assignment。最后还有一种情况，如果某个base class将copy assignment操作符声明为private，编译器将拒绝为其派生类生成一个copy assignment操作符。毕竟编译器为派生类所生成的copy assignment操作符想象中可以处理base class成分。

##### 6. 若不想使用编译器自动生成的函数，就该明确拒绝

所有编译器产出的函数都是public，为了阻止阻止这些函数被创建出来，可以把copy构造和copy assignment声明为private。

声明为private的方法并不绝对安全，因为成员函数和friend函数还是可以调用private函数。

```C++
class HomeForSale {
public:
    ...

private:
    ...
    HomeForSale(const HomeForSale&); // 没有写参数的名称，参数名称并非必要。
    HomeForSale& operator=(const HomeForSale&); // 只有声明，没有实现
};
```

有了上述class定义，当企图拷贝HomeForSale对象，编译器会阻止，如果不慎在成员函数或friend函数之内这么做，链接器会报错。

将链接期错误转移到编译器是可能的。只要将copy构造和copy assignment操作符声明为private就可以了，但不是在HomeForSale类本身，而是在一个专门为了阻止copy动作而设计的base class内:

```C++
class Uncopyable {
protected:
    Uncopyable() {}
    ~Uncopyable() {}

private:
    Uncopyable(const Uncopyable&);
    Uncopyable& operator=(const Uncopyable&);
};
```

为了阻止HomeForSale对象被拷贝，只要继承Uncopyable即可：

```C++
class HomeForSale : private Uncopyable {
    // HomeForSale不再声明copy构造函数和copy assignment操作符 
};
```

这行的通，因为只要任何人--甚至是成员函数或friend函数尝试拷贝HomeForSale对象，编译器便试着生成一个copy构造函数和一个copy assignment操作符，这些函数的“编译器生成版”会尝试调用其base class的对应兄弟，那些调用会被编译器拒绝。

> 为驳回编译器自动提供的机能，可将相应的成员函数声明为private并且不予实现。使用像Uncopyable这样的base class也是一种做法。

##### 7. 为多态基类声明virtual析构函数

```C++
class TimerKeeper {
public:
    TimerKeeper();
    ~TimerKeeper();
    ...
};

class AtomicClock : public TimerKeeper {...}; // 原子钟
class WaterClock : public TimerKeeper {...}; // 水钟
class WristWatch : public TimerKeeper {...}; // 腕表
```

在上述类的基础上设计Factory函数，返回指针指向一个计时对象。Factory函数会返回一个base class指针，指向新生成的derived class对象。

```C++
TimerKeeper* getTimerKeeper();
...

TimerKeeper* ptk = getTimerKeeper(); // 从TimerKeeper继承体系获得一个动态分配的对象。
...
delete ptk;
```

上述问题出在getTimerKeeper返回的指针指向一个derived class对象（例如AtomicClock），而那个对象却经由一个base class指针被删除，而base class中的析构函数是non-virtual，在实际执行时通常发生的是对象的derived成分没有被销毁。

如果class不含有vritual函数，通常表示它并不意图被用作一个base class。当class不企图被当作base class，令其析构函数为virtual往往是个馊主意。考虑如下用来表示二维空间点坐标的class:

```C++
class Point { // 一个二维空间点
public:
    Point(int xCoord, int yCoord);
    ~Point();

private:
    int x, y;
};
```

如果int占用32bits，那么Point对象可塞入一个64bit缓存器中。更有甚者，这样一个Point对象可被当作一个“64 bit”量传给以其他语言如C或FORTRAN撰写的函数。然而当Point的析构函数是virtual，形势起了变化。

欲实现virtual函数，对象必须携带某些信息，主要用来在运行期决定哪一个virtual函数该被调用。这份信息通常是有一个vptr(virtual table pointer)指针指出。vptr指向一个由函数指针构成的数组，称为vtbl(virtual table);每一个带有virtual函数的class都有一个相应的虚函数表。当对象调用某一virtual函数，实际被调用的函数取决于该对象的vptr所指的那个vtbl-编译器在其中寻找适当的函数指针。

如果Point class内含virtual函数，其对象的体积会增加：在32bit计算机体系结构中将占用64bits至96bits(两个int + vptr)。因此，为Point添加一个vptr会增加其对象大小。Point对象不能够塞入一个64-bit缓存器，而C++的Point对象也不再和其他语言（如C）内的相同声明有着一样的结构（因为其他语言的对应物并没有vptr），因此也就不能再把它传递至（或接收自）其他语言所写的函数，除非明确补偿vptr。**只有当class内含至少一个virtual函数，才为他声明virtual析够函数**。

即使class完全不带virtual函数，被“non-virtual析够函数"咬伤还是有可能的。比如：标准string不含任何virtual函数，但有时候程序员会错误地把它当作base class：

```C++
class SpecialString : public std::string {
    ...
};
```

如果在程序的某处无意间将一个point-to-SpecialString转换为一个point-to-string，然后将转换所得的string指针delete，将会产生“行为不明确”。

有时候令class带一个pure virtual析构函数可能颇为有利。pure virtual函数导致abstract classes不能被实体化。然而有时候希望拥有抽象class，但是手上没有任何纯虚函数，可以为class声明一个纯虚析构函数。

```C++
class AWOV {
public:
    virtual ~AWOV() = 0;
};
```

此时必须为这个纯虚析构函数提供一份定义:

```C++
AWOV::~AWOV() {}
```

> - 带多态性质的base classes应该声明一个virtual析构函数。如果class带有任何virtual函数，它就应该拥有一个virtual析构函数。
> - Class设计的目的如果不是作为base class使用，或不是为了具备多态性，就不应该声明virtual析构函数。

##### 8. 别让异常逃离析构函数

```C++
class Widget {
public:
    ...
    ~Widget() {...} // 假设这个可能吐出一个异常
};

void doSomething()
{
    std::vector<Widget> v;
    ...
} // v在这里被自动销毁
```

假设v内含十个Widget，而在析构第一个元素期间，有个异常被抛出。其它9个Widgets还是应该被销毁。

如果析构函数必须执行一个动作，而该动作可能会在失败时抛出异常，该怎么办？假设使用一个class负责数据库连接：

```C++
class DBConnection {
public:
    ...
    static DBConnection create(); // 这个函数返回DBconnection对象

    void close(); // 关闭连接，失败时则抛出异常
};
```

为确保用户不忘记在DBconnection对象身上调用close(),创建一个用来管理DBconnection资源的class，并在其析构函数中调用close()：

```C++
class DBConn {
public:
    ~DBConn()
    {
        db.close();
    }

private:
    DBConnection db;
};

{
    DBConn dbc(DBConnection::create());
    ...
};
```

close失败会抛出异常。

以下两个方法可以避免这一问题。DBConn的析构函数可以：

- 如果close抛出异常就结束程序。通常通过调用abort完成：

```C++
DBConn::~DBConn()
{
    try { db.close(); }
    catch (...) {
        ... // 制作运转log，记下对close的调用失败
        std::abort();
    }
}
```

- 吞下因调用close而发生的异常

```C++
DBConn::~DBConn()
{
    try { db.close(); }
    catch (...) {
        ... // 制作运转log，记下对close的调用失败
    }
}
```

上述两种方法都不好，问题在于两者都无法对“导致close抛出异常”的情况作出反应。较佳的方法如下：

```C++
class DBConn {
public:
    ...
    void close()
    {
        db.close();
        closed = true;
    }

    ~DBConn()
    {
        if (!closed) {
            try {
                db.close();
            }
            catch (...) {
                ... //记录log
            }
        }
    }

private:
    DBConnection db;
    bool closed;
}
```

如果某个操作可能在失败时抛出异常，而又存在某种需要必须处理该异常，那么这个异常必须来自析构函数以外的某个函数。因为析构函数抛出异常就是危险。

> - 析构函数绝对不要抛出异常。如果一个被析构函数调用的函数可能抛出异常，析构函数应该捕获异常，然后吞下他们或结束程序。
> - 如果用户需要对某个操作运行期间抛出的异常作出反应，那么class应该提供一个普通函数（而非析构函数）执行该操作。

##### 9. 绝不在构造和析构过程中调用virtual函数

**不该在构造和析构函数期间调用virtual函数**因为这样的调用不会带来预想的结果。

假设有个class继承体系，用来模拟股市交易如买进、卖出的订单等。这样的交易一定要经过审计，所以每创建一个对象，在审计日志中也需要创建一笔适当记录。如下：

```C++
class Transaction { // 所有交易的base class
public:
    Transaction();
    // 做出一份因类型不同而不同的日志记录
    virtual void logTransaction() const = 0;
    ...
};

Transaction::Transaction() // base class 构造函数实现
{
    ...
    logTransaction(); // 最后动作是用日志记录这笔交易
}

class BuyTransaction : public Transaction { // derived class
public:
    virtual void logTransaction() const;
    ...
};

class SellTransaction : public Transaction { // derived class
public:
    virtual void logTransaction() const;
    ...
};

// 以下操作被执行，会发生什么？
BuyTransaction b;
```

无疑会有一个BuyTransaction构造函数被调用，但首先Transaction构造函数会更早的调用。Transaction构造函数的最后一行调用virtual函数logTransaction.这时候调用的logTransaction是Transaction内的版本。base class构造期间virtual函数绝不会下降到derived classes阶层。**在base class构造期间，virtual函数不是virtual函数**。base class构造函数更早于derived class构造函数， 当base class构造函数执行时derived class的成员变量尚未初始化。

更根本的原因是：在derived class对象的base class构造期间，对象的类型是base class而不是derived class。不只virtual函数会被编译器解析至base class，若使用运行期类型信息如：dynamic_cast和typeid，也会把对象视为base class类型。

相同道理也适用于析构函数，一旦derived class析构函数开始执行，对象内的derived class成员变量便呈现未定义值，进入base class析构函数后就成为一个base class对象。而C++的virtual 函数，dynamic_cast等等也是这么认为。

解决方法：一种做法是在class Transaction内将logTransaction函数改为non-virtual，然后要求派生类构造函数传递必要信息给base class构造函数。

```C++
class Transaction {
public:
    explict Transaction(const std::string& logInfo);
    void logTransaction(const std::string& logInfo) const; // 改为non-virtual函数
    ...
};

Transaction::Transaction(const std::string& logInfo) {
    ...
    logTransaction(logInfo); // non-virtual 调用
}

class BuyTransaction : public Transaction {
public:
    BuyTransaction(params)
    : Transaction(createLogString(params)) // 将log信息传递给base class 构造函数
    {...}

private:
    static std::string createLogString(params);
};
```

**由于无法使用virtual函数从base class向下调用，在构造期间，可以籍由“令derived class将必要的构造信息向上传递至base class构造函数”替换而加以弥补。**

static成员函数createLogString的使用比起直接在成员初始化列表内给予base class所需数据。利用辅助函数创建一个值传给base class往往比较方便，令此函数为static，也就不可能意外指向”未初始化的Buytransaction 成员变量“。

> - 在构造和析构期间不要调用virtual函数，因为这类调用从来不会下降到derived class

##### 10. 令 operator= 返回一个 reference to *this

关于赋值，可以把他们写成连锁形式：

```C++
int x, y, z;
x = y = z = 15;
```

为了实现”连锁赋值“，赋值操作符必须返回一个reference指向操作符的左侧实参。这是为class实现赋值操作符时应该遵循的协议：

```C++
class Widget {
public:
    ...
    Widget& operator=(const Widget& rhs) { // 返回类型是个reference指向当前对象。
        ...
        return *this; // 返回左侧对象
    }
};
```

这个协议不仅适用于以上的标准赋值形式，也适用于所有赋值相关运算，例如：

```C++
class Widget {
public:
    ...
    Widget& operator+=(const Widget& rhs) { // 适用于+=， -=， *=,等
        ...
        return *this;
    }

    Widget&& operator=(int rhs) { // 此函数也适用，即使此操作符的参数类型不符协定。
        ...
        return *this;
    }
};
```

> - 令赋值操作符返回一个reference to *this.

##### 11. 在operator= 中处理”自我赋值“（内容不太理解，看完后续章节再看）

”自我赋值“发生在对象被赋值给自己时：

```C++
class Widget {...};
Widget w;
...
w = w; // 赋值给自己
```

虽然看起来有些蠢，但是合法。此外赋值动作并不总是可以被一眼辨识出来，例如：

```C++
a[i] = a[j]; // 潜在的自我赋值
*px = *py // 潜在的自我赋值
```

```C++
class Base { ... };
class Derived : public Base { ... };
void doSomething(const Base& rb, Derived* pd); // rb 和 *pd有可能其实是同一个对象
```

如果尝试自行管理资源（见后续的资源管理），可能会掉进”在停止使用资源之前意外释放了它“的陷阱。假设建立一个class用来保存一个指针指向一块动态分配的位图（bitmap):

```C++
class Bitmap { ... };
class Widget {
    ...
private:
    Bitmap* pb; // 指向一个从heap分配而得的对象
}
```

下面是operator= 实现代码，表面上看起来合理，但自我赋值出现时并不安全（也不具备异常安全性）：

```C++
Widget& Widget::operator=(const Widget& rhs) { // 不安全的实现版本
    delete pb; // 停止使用当前的bitmap
    pb = new Bitmap(*rhs.pb); // 使用rhs‘s bitmap的副本
    return *this;
}
```

这里的自我赋值的问题是，operator=函数内的*this和rhs有可能是同一个对象。如果是这样delete就是删除的rhs的bitmap。然后new Bitmap参数使用的是一个已经释放的对象。

欲阻止这种错误，传统的做法是在operator=最前面进行”证同测试“达到”自我赋值“的检验目的：

```C++
Widget& Widget::operator=(const Widget& rhs) {
    if (this == &rhs) return *this; // 证同测试

    delete pb;
    pb = new Bitmap(*rhs.pb);
    return *this;
}
```

这个新版本仍然存在异常方面的麻烦：如果”new Bitmap"导致异常（不论是因为分配时内存不足或因为Bitmap的copy构造函数抛出异常），Widget最终会持有一个指针指向一块被删除的Bitmap。这样的指针有害。无法安全的删除他们。

让operator= 具备“异常安全性”往往自动获得“自我赋值安全”。例如下面的代码，我们只需要注意在复制pb所指东西之前别删除pb;

```C++
Widget& Widget::operator= (const Widget& rhs) {
    Bitmap* pOrig = pb; // 记住原先的pb
    pb = new Bitmap(*rhs.pb);
    delete pOrig;
    return *this;
}
```

现在，如果“new Bitmap"抛出异常， pb(及其栖身的那个Widget)保持原状。即使没有证同测试，这段代码还是能够处理自我赋值，因为我们对原bitmap做了一份复件，删除原bitmap，然后指向新制造的那个复件。

在operator=函数内手工排列语句（确保代码不但”异常安全“而且”自我赋值安全“）的一个替代方案是，使用所谓的copy and swap技术。这个技术和“异常安全性”有密切关系，由条款29详细说明：

```C++
class Widget {
    ...
    void swap(Widget& rhs); // 交换rhs和*this的数据，详见条款29
    ...
};

Widget& Widget::operator=(const Widget& rhs) {
    Widget temp(rhs); // 为rhs数据制作一份复件
    swap(temp); // 将*this数据和上述复件的数据交换
    return *this;
}
```

这个主题的另外一种方法是利用以下事实:(1) 某class的copy assignment操作符可能被声明为“以by value方式接受实参”;(2)以by value方式传递东西会造成一份复件（见条款20）：

```C++
Widget& Widget::operator= (Widget rhs) // rhs是被传对象的一份复件
{
    swap(rhs);
    return *this;
}
```

> - 确保当对象自我赋值时operator= 有良好的行为。其中技术包括比较“来源对象”和“目标对象”的地址，精心周到的语句顺序，以及copy-and-swap。
> - 确定任何函数如果操作一个以上的对象，而其中多个对象是同一个对象时，其行为仍然正确。

##### 12. 复制对象时勿忘其每一个成分

考虑一个class用来表现顾客，其中手工写出copying 函数，使得外界对它们的调用会被日志记录下来：

```C++
void logCall(const std::string& funcName);
class Customer {
public:
    ...
    Customer(const Customer& rhs);
    Customer& operator=(const Customer& rhs);
    ...
private:
    std::string name;
};

Customer::Customer(const Customer& rhs)
: name(rhs.name)
{
    logCall("Cunstomer copy constructor");
}

Customer& Customer::operator= (const Cunstomer& rhs)
{
    logCall("Customer copy assignment operator");
    name = rhs.name;
    return *this;
}
```

这里一切看起来都很好，直到另一个成员变量加入：

```C++
class Date {...};
class Customer {
public:
    ... // 同前
private:
    std::string name;
    Date lastTransaction;
};
```

这时候既有的copying函数执行的是局部拷贝，如果为class添加一个成员变量，必须同时修改copying函数。

一旦发生继承，可能会造成潜藏危机：

```C++
class PriorityCustomer : public Customer {
public:
    ...
    PriorityCustomer(const PriorityCustomer& rhs);
    PriorityCustomer& operator= (const PriorityCustomer& rhs);
    ...
private:
    int priority;
};

PriorityCustomer::PriorityCustomer(const PriorityCustomer& rhs)
: priority(rhs.priority)
{
    logCall("PriorityCustomer copy constructor");
}

PriorityCustomer& PriorityCustomer::operator=(const PriorityCustomer& rhs)
{
    logCall("PriorityCustomer copy assignment operator");
    priority = rhs.priority;
    return *this;
}
```

PriorityCustomer的copying函数看起来好像复制了PriorityCustomer内的每一样东西，但是再看一眼。它们没有复制在base class中声明的成员变量。

任何时候为derived class编写copying函数，必须很小心地也复制其base class部分，那些成分往往是private，所以无法直接访问他们，应该让derived class的copying函数调用base class的函数：

```C++
PriorityCustomer::PriorityCustomer(const PriorityCustomer& rhs)
: Customer(rhs) // 调用base class copy构造函数
, priority(rhs.priority)
{
    logCall("PriorityCustomer copy constructor");
}

PriorityCustomer&
PriorityCustomer::operator= (const PriorityCustomer& rhs)
{
    logCall("PriorityCustomer copy assignment operator");
    Customer::operator=(rhs); // 对base class成分进行赋值动作
    priority = rhs.priority;
    return *this;
}
```

> - Copying函数应该确保复制“对象内的所有成员变量”及“所有base class 成分”。
> - 不要尝试以某个copying函数实现另一个copying函数。应该将共同机能放进第三个函数中，并且由两个copying函数共同调用。