---
layout: post
title: 分享一个调试技巧(C\C++)
---

**TL;DR** C语言中宏的黑魔法

在调试代码时，有时会遇到报错的的函数是没问题，而是调用它的函数传入的参数是异常的，而又没法通过代码准确定位到具体调用的位置的情况，特别是对一些基础函数，往往会出现这种情况，因为工程中调用它们的地方太多，而且有时会漏掉返回值检测。这里给出一个快速找出调用者是谁的方法。

```
//filename: tt.h
#include <stdio.h>

int foo(int p);

#define foo(_a) \
    (printf("[%s:%d]call foo()\n", __FUNCTION__, __LINE__), foo(_a))
```

[2012/09/17]用上面的宏替代了下面的宏，下面的的宏不能正确返回函数返回值：

```
//#define foo(_a) \
//    do { \
//        printf("[%s:%d]call foo()\n", __FUNCTION__, __LINE__); \
//        foo(_a); \
//    } while (0)
```


```
//filename: tt.c
#include "tt.h"

#ifdef foo
#undef foo
#endif

int foo(int p)
{
    printf("input = %d\n", p);
    return p;
}

//filename: main.c
#include "tt.h"

int main()
{
    printf("return %d", foo(1024));
    return 0;
}
```

未定义`#define foo(_a)`宏时执行结果如下（将tt.h中定义foo的那段代码注释掉）：

```
$ cc main.c tt.c
$ ./a.out

>> input = 1024
```

定义了`#define foo(_a)`宏之后执行结果如下：

```
$ cc main.c tt.c
$ ./a.out

>> [main:5]call foo()
>> input = 1024
```

通过定义与函数一致的宏，调用的时候执行了修改了之后的代码，这样我们就可以在其中添加一些我们需要的信息，方便我们调试。
