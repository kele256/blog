# GObject
## 类结构体
类似于C++中，属于类的部分，比如static成员函数、static成员变量等
## 实例结构体
类似于C++中，每个实例独有的内容，比如成员函数，成员变量等。

## 定义类型函数
```
GType xx_xx_get_type(void)
```

## 使用GObject库模拟基于类的数据封装
- 在头文件中包含glib-object.h
- 在头文件中构建实例结构体和类结构体，并分别将GObject类的实例结构与类结构置于成员之首
- 在头文件中定义P_TYPE_T宏，并声明p_t_get_type()函数
- 在.c文件中调用G_DEFINE_TYPE宏产生类型注册代码

## 私有属性模拟
- 定义PM_DLIST_GET_PRIVATE宏
- 在class_init中，注册私有变量

## .h的实现
```C++
#ifndef PM_DLIST_H
#define PM_DLIST_H

#include <glib-object.h>

// 定义子类类型，其中pm_dlist是用户可以修改的内容，表示这个类的用途，后续所有的函数都必须以pm_dlist开头。
#define PM_TYPE_DLIST (pm_dlist_get_type())

// 实例结构体，类似于C++实例化时，每个实例独有的内容，比如成员函数，成员变量等
typedef struct _PMDList PMDList;
struct _PMDList {
    GObject parent_instance;
};

// 类结构体，类似于C++中，属于类的部分，比如：static成员函数和static成员变量等
typedef struct _PMDListClass PMDListClass;
struct _PMDListClass {
    GObjectClass parent_class;
};

// private, 定义获取类型函数，供PM_TYPE_DLIST调用，不建议直接调用
GType pm_dlist_get_type(void);

#endif
```

## .c的实现
```C++
#include "pm-dlist.h"

// 此宏有三个功能
// 1. 函数GType pm_dlist_get_type(voide)的实现，GObject帮我们具体实现
// 2. 注册函数前缀pm_dlist到GObject
// 3. 注册实例结构体名称PMDList到GObject
// 通过此宏的调用，GObject知道了我们的三个信息：类型，类名称，函数前缀
G_DEFINE_TYPE(PMDList, pm_dlist, G_TYPE_OBJECT);

// 将私有成员PMDListPrivate注册到GObject中，后续通过宏PM_DLIST_GET_PRIVATE作为获取私有成员变量的唯一入口，如果需要生效，需要在class init中注册此变量
#define PM_DLIST_GET_PRIVATE(obj) (\
        G_TYPE_INSTANCE_GET_PRIVATE((obj), PM_TYPE_DLIST, PMDListPrivate))

// 普通结构体，作为实例结构体的私有成员
typedef struct _PMDListNode PMDListNode;
struct _PMDListNode {
    PMDListNode *prev;
    PMDListNode *next;
    void *data;
};

// 普通结构体，作为实例结构体的私有成员
typedef struct _PMDListPrivate PMDListPrivate;
struct _PMDListPrivate {
    PMDListNode *head;
    PMDListNode *tail;
};

// 类结构体的初始化，类似于C++中的static成员变量的初始化，函数地址分配等，此函数只会在第一次类实例化的时候被调用，后续类实例化不再调用。
static void
pm_dlist_class_init(PMDListClass* kclass)
{
    // 成员函数定义，将PMDListPrivate定义为类成员变量
    g_type_class_add_private(kclass, sizeof(PMDListPrivate));
}

// 实例结构体的初始化，类似于C++中的构造函数
static void
pm_dlist_init(PMDList* self)
{
    PMDListPrivate *priv = PMD_DLIST_GET_PRIVATE(self);

    priv->head = NULL:
    priv->tail = NULL;
}

#endif
```
## 子类私有属性的外部访问
- 实现p_t_set_property与p_t_get_property函数，让他们来完成g_object_new函数的“属性名-属性值”结构向GObject子类属性的映射。
- 在GObject子类的类结构体初始化函数中，让GObject类（基类）的两个函数指针set_property与get_property分别指向p_t_set_property与p_t_get_property函数。
- 在GObject子类的类结构体初始化函数中，为GObject子类安装属性

前两个步骤，可以理解为GObject的两个虚函数的实现。

## GObject子类对象的析构过程
- dispose
- finalize
### GObject 引用计数与引用循环
- 使用g_object_new函数进行对象实例化的时候，对象的引用计数为1；
- 每次使用g_object_ref函数引用对象时，对象的引用计数便会增1；
- 每次使用g_object_unref函数为对象解除引用时，对象的引用计数便会减1；
- 在g_object_unref函数中，如果发现对象的引用计数为0，那么则调用对象的析构函数释放对象所占用的资源；

GObject类及其子类对象不仅存在继承关系，还存在相互包含的关系，
