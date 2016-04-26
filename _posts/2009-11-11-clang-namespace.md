---
layout: post
title: C语言的作用域/namespace分析
---

在csdn上看到一段代码。觉得很有意思，于是便自己动动手分析分析。这是用于分析C语言中的作用的一段代码，值得研究研究。

*Demo代码中`calloc`之后并没有`free`掉。*

好吧，我们从代码开始。

原始代码：

```
#include <stdio.h>
#include <stdlib.h>

int x(const int int_a) {return int_a;}

struct x
{
    int x;
};

#define x(x)  x

int main(int argc, char *argv[])
{
    int *x = calloc(1, sizeof x);
    x: (((struct x *)x)->x) = x(5);
    printf("%p\n", ((struct x *)x)->x);
    return 0;
}
/*
output:
0x5
*/
```

[0] 变量名(包括指针名, 函数名)和自定义类型名(struct)存在于不同`namespace`. 所以`b`不会和`a`, `c`冲突

```
int x(const int int_a) {return int_a;}      //a

struct x                                    //b
{
    int x;
};

#define x(x)  x

int main(int argc, char *argv[])
{
    int *x = calloc(1, sizeof x);           //c
    x: (((struct x *)x)->x) = x(5);
    printf("%p\n", ((struct x *)x)->x);
    return 0;
}
```

[1] `(int *)x`和`(int *(const int))x`不在同一层`namespace`, 编译通过.
链接时出错. `(int *)x`将`(int *(const int))x`覆盖, 所以在`c`行时会找不到匹配的函数名

```
int x(const int int_a) {return int_a;}   //a

struct x
{
    int x;
};

//#define x(x)  x

int main(int argc, char *argv[])
{
    int *x = calloc(1, sizeof x);        //b
    x: (((struct x *)x)->x) = x(5);      //c
    printf("%p\n", ((struct x *)x)->x);
    return 0;
}
```

[2] 编译通过

```
int x(const int int_a) {return int_a;}

struct x
{
    int x;
};

//#define x(x)  x

int main(int argc, char *argv[])
{
    int *x = calloc(1, sizeof x);
    //x: (((struct x *)x)->x) = x(5);
    //printf("%p\n", ((struct x *)x)->x);
    return 0;
}
```

[3] 编译出错. `(int *)x`和`(int *(const int))x`在同一层`namespace`, 冲突.

```
int x(const int int_a) {return int_a;}

int *x;

int main(int argc, char *argv[])
{
    return 0;
}
```

[4] `#define`在预编译阶段替换其后面代码, 所以对`#define`后面的代码来说, `x(n)`被替换为`n`, 所以在编译时代码会扩展为如下:

```
int x(const int int_a) {return int_a;}

struct x
{
    int x;
};

#define x(x)  x

int main(int argc, char *argv[])
{
    int *x = calloc(1, sizeof x);
    x: (((struct x *)x)->x) = 5;         //x(5)替换为5,所以并没有调用函数int x(const int)
    printf("%p\n", ((struct x *)x)->x);
    return 0;
}
```

关于标签`label`.

标签, 仅仅是一个符号, 存在一个`label`专用的`namespace`, 仅对`goto`可见, 所以不会与变量或者常量冲突.

在MSDN的`goto`条目中也有相关的描述:

> The set of identifier names following a goto has its own name space so the names do not interfere with other identifiers. Labels cannot be redeclared.

```
int main(int argc, char *argv[])
{
    int p = 5;
    x: printf("line x.\n");
    ++p;
    if (p == 8) return 0;
    goto x;
}

int main(int argc, char *argv[])
{
    int x = 5;
    x: printf("label x.\n");          //这里的label x与(int)x是无关的
    ++x;
    if (x == 8) return 0;
    goto x;                          //goto会在标签namespace查找label x.
    printf("x = %d | :( .\n", x); //无效语句
    return 0;                     //无效语句
}
```

[5] 所以代码中`label x`与其他命名不冲突

```
int x(const int int_a) {return int_a;}

struct x
{
    int x;
};

#define x(x)  x

int main(int argc, char *argv[])
{
    int *x = calloc(1, sizeof x);
    x: (((struct x *)x)->x) = x(5);         //这里的label x存在于独立的namespace,与其他不冲突.
    printf("%p\n", ((struct x *)x)->x);
    return 0;
}
```

[6] 现在我们把代码中无效代码去掉, 并把宏定义语句手动替换掉, 是的代码简洁点

```
struct x
{
    int x;
};

int main(int argc, char *argv[])
{
    int *x = calloc(1, sizeof(int *));
    //到此我们有自定义类型struct x和变量(int *)x,其中struct x作用域为全局,(int *)x作用域为main()
    (((struct x *)x)->x) = 5;
    //            ↑
    //           这里的x是由(int *)强制转化成(struct x *),所以后面实际是给struct中的(int)x赋值
    printf("%p\n", ((struct x *)x)->x);   //这里还是需要强制转化成struct,这样才能识别,然后得到(int)x的值
    return 0;
}
```

到此代码中所有的`x`都说明了, 这里再次总结下.

```
#include <stdio.h>
#include <stdlib.h>

int x(const int int_a) {return int_a;}      //全局的函数名

struct x                                    //全局的struct名,属于自定义类型名,所以不会跟上面的(int *(const int))x及下面main中的(int *)x冲突{
    int x;                                  //属于struct x的int型x
};

#define x(x)  x                             //宏定义,会在预编译时进行代码扩展,所以并不会在编译时产生命名冲突

int main(int argc, char *argv[])
{
    int *x = calloc(1, sizeof x);           //作用域为main的(int *)x; sizeof计算的是(int *)x大小.
    x: (((struct x *)x)->x) = x(5);         //作为label的x独立存在于一个namespace.
    printf("%p\n", ((struct x *)x)->x);
    return 0;
}
```
