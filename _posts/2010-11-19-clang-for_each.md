---
layout: post
title: for_each宏
---

<p class="lead">C语言中宏的黑魔法</p>

之前在编码的时候遇到几次需要将这么几个值统一处理下，当时也没有想到什么好招，就每个量都写了相同的代码，一直觉得很土，加之使用python时的for...in...的美好感觉，便写了个for_each的宏。

for_each宏能够很方便遍历一组零散的元素，而且在遍历完之后将不再需要的临时申请的空间释放掉。

set_list_m每次添加一个元素，set_list_f则会一次将所需要遍历的元素全部加入。

因为懒set_list_m宏中应该给变量加括号的也就懒着加了。

    #include <stdio.h>
    #include <stdlib.h>
    #include <string.h>
    #include <stdarg.h>

    struct list_t {
            void                 *value;
            struct list_t         *next;
    };

    #define set_list_m(_p, _list) \
            do { \
                    if (_list == NULL) { \
                            _list = (struct list_t *)malloc(sizeof(struct list_t)); \
                            _list->next = NULL; \
                            _list->value = _p; \
                    } \
                    else { \
                            struct list_t *_tmp = _list; \
                            while (_tmp->next != NULL) { \
                                    _tmp = _tmp->next; \
                            } \
                            _tmp->next = (struct list_t *)malloc(sizeof(struct list_t)); \
                            _tmp = _tmp->next; \
                            _tmp->next = NULL; \
                            _tmp->value = _p; \
                    } \
            } \
            while (0)

    /*
     * 写for_each宏的原由是为了方便遍历一组零散的元素(这些元素并没有通过数组/列表等组织在一起)
     * 而python中的for...in...也不停诱惑着我
     */
    #define for_each(_p, _list) \
            for ( struct list_t *_tmp = (_list), *_front; \
                  (_tmp != NULL) && (((_p) = _tmp->value) || 1); \
                  _front = _tmp, _tmp = _tmp->next, free(_front) )

    /*
     * struct list_t *set_list_f(void *p, ...)
     * 需要要以NULL结束
     * 返回for_each可用的list
     */
    struct list_t *set_list_f(void *p, .../* NULL */)
    {
            va_list arg_ptr;
            va_start(arg_ptr, p);

            struct list_t *_tmpnode = NULL;
            struct list_t *list = NULL;
            void *_tmpval = p;
            while (_tmpval != NULL) {
                    if (_tmpnode == NULL) {
                            _tmpnode = (struct list_t *)malloc(sizeof(struct list_t));
                            list = _tmpnode;
                    }
                    else {
                            _tmpnode->next = (struct list_t *)malloc(sizeof(struct list_t));
                            _tmpnode = _tmpnode->next;
                    }
                    _tmpnode->next = NULL;
                    _tmpnode->value = _tmpval;

                    _tmpval = va_arg(arg_ptr, void *);
            }

            va_end(arg_ptr);

            return list;
    }

    int main()
    {
            //////////////////////////////////////////////////////
            int a = 1, b = 2, c = 3;

            struct list_t *l1 = NULL;
            set_list_m(&a, l1);
            set_list_m(&b, l1);
            set_list_m(&c, l1);

            int *p1 = NULL;
            for_each(p1, l1) {
                    printf("for_each\n");
                    *p1 = 4;
            }

            printf("a = [%d], b = [%d], c = [%d]\n", a, b, c);

            //////////////////////////////////////////////////////
            a = 1, b = 2, c = 3;

            struct list_t *l2 = set_list_f(&a, &b, &c, NULL);

            int *p2 = NULL;
            for_each(p2, l2) {
                    printf("for_each\n");
                    *p2 = 5;
            }

            printf("a = [%d], b = [%d], c = [%d]\n", a, b, c);

            return 0;
    }
