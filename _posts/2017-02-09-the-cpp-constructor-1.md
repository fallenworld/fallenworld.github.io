---
layout: post
title: C++中的构造函数(一)：默认构造函数
categories: C++底层解析
tags: C/C++
---

* content
{:toc}

## 默认构造函数的定义 ##

ISO C++ Standard中有说：

> A default constructor for a class X is a constructor of class X that can be called without an argument.

在C++中，不需要任何参数就可以调用的构造函数称为默认构造函数

注意：这里不仅包含没有任何参数的构造函数，参数具有默认值的也算进去，例如
```c++
A::A();
A::A(int a = 1)
```
上面两个函数都算是默认构造函数
## 编译器合成的默认构造函数 ##

ISO C++ Standard的Section 12.1中：

> If there is no user-declared constructor for class X, a constructor having no parameters is implicitly declared as defaulted . An implicitly-declared default constructor is an inline public member of its class.

如果某个类没有定义任何构造函数的话，那么编译器就会为这个类合成一个默认构造函数，编译器合成的这个默认构造函数是public inline的

由于这个被编译器合成出来的默认构造函数是一个内联函数，因此在目标文件的符号表中不会有这个函数的符号，调用这个函数也是通过把代码直接插进去的方式，而不是通过call指令

## trival的默认构造函数 ##
一般情况下合成的这个默认构造函数是一个空的函数，什么也都不做，称作trival的（trival翻译：浅薄的、无价值的），例如：

```c++
class A
{
    int a;
};
```
这时候编译器为A合成的默认构造函数类似是这样的：
```c++
//c++伪代码
inline A::A()
{}
```
由于此时这个函数是一个空的内联函数，因此调用这个函数实际上相当于什么也没做，没有任何额外的代码需要被插入到调用处。

但是注意：如果你自己编写一个空的什么也不做的默认构造函数的话，这个你自己写的默认构造函数不会被认为是trival的，编译器照样会为这个函数生成代码，并且在符号表里也有这个函数。也就是说只有编译器自动合成的默认构造函数才可能是trival的

## 非trival的默认构造函数 ##
有三种特殊情况编译器需要为自动合成的默认构造函数生成实际的代码，此时编译器生成的这个默认构造函数就不是trival的了，因为此时这个合成的默认构造函数是包含一些实际的代码的

C++ Standard中说明了这三种情况：（标准中说的是trival的情况，反过来就是非trival的情况）

> A default constructor is trivial if it is not user-provided and if:
> 
> — its class has no virtual functions and no virtual base classes, and
> 
> — no non-static data member of its class has a brace-or-equal-initializer, and
>
> — all the direct base classes of its class have trivial default constructors, and
>
> — for all the non-static data members of its class that are of class type (or array thereof), each such class has a trivial default constructor.

这三种情况分别是：
- 包含虚函数或继承自虚基类
- 类内非静态成员初始化
- 包含类成员或继承自父类，且这些类中有默认构造函数不是trival的（标准中的最后两种情况合起就来是第三种）

以上这三种情况实际上就是[C++中的构造函数(二)：构造函数代码扩张](http://blog.fallenworld.org/2017/02/22/the-cpp-constructor-2/)中所说的四种需要进行构造函数代码扩张的情况中的三种

## 参考资料 ##

《深度探索C++对象模型》 2.1节 

C++ Standard（ISO/IEC 14882-2011） 12.1节
