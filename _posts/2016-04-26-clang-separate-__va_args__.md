---
layout: post
title: 分离C语言中的可变宏参数__VA_ARGS__
---

**TL;DR** C语言中宏的黑魔法

最近项目中遇到需要针对可变宏参数`__VA_ARGS__`中的每个参数进行**宏处理**的情况. Google了下没有找到, 便自己琢磨实现了一个.

这里用到了获取`__VA_ARGS__`个数的辅助宏`ARGS_SIZE`.

```
#include <stdio.h>

/**
 * max args size: 9
 */
#define GET_A10(a1, a2, a3, a4, a5, a6, a7, a8, a9, a10, ...)     a10
#define ARGS_SIZE(...)          GET_A10(__VA_ARGS__, 9, 8, 7, 6, 5, 4, 3, 2, 1, throwaway)

#define FOREACH(...)            SEPARATE(ARGS_SIZE(__VA_ARGS__), __VA_ARGS__)
#define SEPARATE_(n)            SEPARATE_##n
#define SEPARATE(n, ...)        SEPARATE_(n)(__VA_ARGS__)

#define SEPARATE_1(arg)         DO(arg)
#define SEPARATE_2(arg, ...)    DO(arg); SEPARATE_1(__VA_ARGS__)
#define SEPARATE_3(arg, ...)    DO(arg); SEPARATE_2(__VA_ARGS__)
#define SEPARATE_4(arg, ...)    DO(arg); SEPARATE_3(__VA_ARGS__)
#define SEPARATE_5(arg, ...)    DO(arg); SEPARATE_4(__VA_ARGS__)
#define SEPARATE_6(arg, ...)    DO(arg); SEPARATE_5(__VA_ARGS__)
#define SEPARATE_7(arg, ...)    DO(arg); SEPARATE_6(__VA_ARGS__)
#define SEPARATE_8(arg, ...)    DO(arg); SEPARATE_7(__VA_ARGS__)
#define SEPARATE_9(arg, ...)    DO(arg); SEPARATE_8(__VA_ARGS__)

#define DO(arg)                 printf("LOG: " arg "\n")

int main()
{
    FOREACH("s0", "s1", "s2", "s3", "s4", "s5", "s6", "s7", "s8");
    return 0;
}
```
