---
layout: post
title: C++ Template中防止使用不完整类型
---

Chromium的源码中有这么一段代码，看到两行奇怪的代码（?处），对于使用C++模板的人来说应该是多加注意并学习的。

    //filename: scoped_ptr.h
    template <class C>
    class scoped_ptr {
     public:
      explicit scoped_ptr(C* p = NULL) : ptr_(p) { }

      ~scoped_ptr() {
        enum { type_must_be_complete = sizeof(C) };     //?
        delete ptr_;
      }

      void reset(C* p = NULL) {
        if (p != ptr_) {
          enum { type_must_be_complete = sizeof(C) };   //?
          delete ptr_;
          ptr_ = p;
        }
      }

     private:
      C* ptr_;

      //...
    };

这里我们简化一下代码，并作一下对比，就能知道其中的缘由了。

    //Code 1
    template<class T>
    class A {
    private:
        int i;
    };

    int main()
    {
        A<int> a;
        return 0;
    }

    >>编译无异常

------

    //Code 2
    class B;

    template<class T>
    class A {
    private:
        int i;
    };

    int main()
    {
        A<B> a;
        return 0;
    }

    >>编译无异常，但是这却是有问题的，因为B的类型不明确

------

    //Code 3
    class B;

    template<class T>
    class A {
    public:
        A() { enum{x=sizeof(B)}; }
    private:
        int i;
    };

    int main()
    {
        A<B> a;
        return 0;
    }

    >>编译出错
    >>tt.cpp: In constructor ‘A<T>::A()’:
    >>tt.cpp:6: error: invalid application of ‘sizeof’ to incomplete type ‘B’
