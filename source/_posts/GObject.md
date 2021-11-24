# GObject
GObject是一个程序库，可以帮助我们使用C语言编写面向对象程序。

``` C
#include <glib-object.h>

typedef struct _PMDListNode PMDlistNode;
struct _PMDListNode
{
    PMDListNode *prev;
    PMDListNode *next;
    void* data;
};

typedef struct _PMDList PMDList;

struct 
```