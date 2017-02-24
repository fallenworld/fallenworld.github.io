---
layout: post
title: C语言中声明的语法/语义详解
categories: C/C++
tags: C/C++
---

* content
{:toc}

## 前言 ##

有时候我们会看到C语言中有非常复杂的声明语句，尤其是把指针、数组、函数、限定符(限定符例如const、volitale)结合起来的时候，比如：

```c
int (*a)[6];    //指向数组的指针
int (*fun[3])(int x, int y);    //函数指针的数组
```

看到这样复杂的声明的时候，很多人经常会无从下手，无法理解这个声明的含义

于是接下来我来从语法和语义的角度来详细分析一下声明语句中指针、数组、函数的用法

## 声明的语法定义 ##
C Standard文档定义了声明的语法和语义，我们就从标准文档入手，ISO C90中关于声明的语法定义如下：  
（理解以下内容需要一些编译原理的知识）
>declaration := declaration-specifiers init-declarator-list(opt) ;  
>declaration-specifiers :=   
>&emsp;&emsp;storage-class-specifier declaration-specifiers(opt) |   
>&emsp;&emsp;type-specifier declaration-specifiers(opt) |  
>&emsp;&emsp;type-qualifier declaration-specifiers(opt) |  
>&emsp;&emsp;function-specifier declaration-specifiers(opt)  
>init-declarator-list := init-declarator |  init-declarator-list , init-declarator  
>init-declarator := declarator | declarator = initializer

（注：上面语法描述中的的(opt)是在指在这个产生式中这一个非终结符是可选的，例如S := A B(opt);中B是可有可无的）

我们抽取出我们所关注的语法部分（这里不管初始化的语法），可知声明的语法定义如下：  

声明 := 说明符 声明符列表  
声明符列表 := 声明符 | 声明符列表, 声明符

由上可知一个声明是由两个部分组成，前面是说明符，后面是声明符列表。那么什么是说明符和声明符列表呢？举个例子：  
```c
static const int a, *b, c[6];  
```
其中前面的static const int就是说明符，而a, \*b, c[6]这三者整体就是声明符列表。  
所谓说明符：就是由声明语句前面的像static、int之类的关键字所组成的  
而声明符列表：就是声明语句中除去说明符以外的部分，它是一个或多个由逗号所分割的声明符

那么声明符又是什么呢？在上例中，a和*b以及c[6]这三个都是声明符，C Standard中声明符的语法定义如下：
>declarator := pointer(opt) direct-declarator  
>direct-declarator :=  
>&emsp;&emsp;identifier |  
>&emsp;&emsp;( declarator ) |  
>&emsp;&emsp;direct-declarator [ constant-expression(opt) ] |  
>&emsp;&emsp;direct-declarator ( parameter-type-list ) |  
>&emsp;&emsp;direct-declarator ( identifier-list(opt) )  
>pointer := \* type-qualifier-list(opt) | * type-qualifier-list(opt) pointer  
>type-qualifier-list := type-qualifier | type-qualifier-list type-qualifier  
>parameter-type-list := parameter-list | parameter-list, ...  
>parameter-list := parameter-declaration | parameter-list , parameter-declaration  
>parameter-declaration :=  
>&emsp;&emsp;declaration-specifiers declarator  |  
>&emsp;&emsp;declaration-specifiers abstract-declarator(opt)  
>identifier-list := identifier | identifier-list , identifier

提取出核心的语法定义就是：

声明符 := 直接声明符 | 指针符 直接声明符  
直接声明符 :=  
&emsp;&emsp;标识符 |  
&emsp;&emsp;(声明符) |  
&emsp;&emsp;直接声明符[常量表达式] |  
&emsp;&emsp;直接声明符(参数表) |  
&emsp;&emsp;直接声明符(标识符表(opt))  
指针符 := \* 类型限定词列表(opt) | \* 类型限定词列表(opt) 指针符

由上面的分析可知声明语句中核心的部分就是声明符，只要能理解好声明符就能理解这个声明语句的含义

## 声明的语义 ##

接下来我们就分四类来解析声明的语义

### 最简单的声明语句的语义 ###
C Standard 6.7.5中：
>If, in the declaration ‘‘T D1’’, D1 has the form  
>&emsp;identifier  
>then the type specified for ident is T .  
> If, in the declaration ‘‘T D1’’, D1 has the form  
>&emsp;( D )  
>then ident has the type specified by the declaration ‘‘T D’’. Thus, a declarator inparentheses is identical to the unparenthesized declarator

最简单的声明语句就是形如这样的：

```c
const int a;      //a是const int类型
const int (b);    //和上面的等价
```
首先形如T D;且T是说明符且D是标识符，这样的最简单的声明中标识符的类型就是T，例如int a;这样最简单的就符合这个形式

其次由标准可知整个声明符加括号和不加括号的两种声明方式是等价的，例如上例中a和b的类型是相同的

### 指针的语义 ###
和指针相关的语法如下：

声明符 := 指针符 直接声明符    
指针符 := \* 类型限定词列表(opt) | \* 类型限定词列表(opt) 指针符  
类型限定词列表 := 类型限定词 | 类型限定词列表 类型限定词  
类型限定词 := const | restrict | volatile

由上面的语法可知，"\*"后面所紧接着的const、volatile、restrict都是和"\*"是一个整体的，它们组成了指针符

而关于指针的语义，C Standard 6.7.5.1中：

>If, in the declaration ‘‘T D1’’, D1 has the form  
>&emsp;\* type-qualifier-list(opt) D  
>and the type specified for ident in the declaration ‘‘T D’’ is ‘‘derived-declarator-type-list T ’’, then the type specified for ident is ‘‘derived-declarator-type-list type-qualifier-listpointer to T ’’. For each type qualifier in the list, ident is a so-qualified pointer.

怎么理解上面这段话呢？简单来说就是我们可以把所有的指针声明归结成  
&emsp;T \* 类型限定词表 D;  
的形式。  
而想要知道    
&emsp;T \* 类型限定词表 D;  
这样子的一个声明语句中标识符的类型的话，我们要先知道  
&emsp;T D;   
这个声明语句中标识符的类型。

C Standard中规定：若T D;中标识符的类型是T1的话，则T \* 类型限定词表 D;中标识符的类型是"指向T1的指针且受类型限定词列表的限定"（注意T1不一定就是T）。因此我们要先分析去掉指针符的声明。举个例子，现在有这样一个声明：
```c
const int * const volatile a;
```
我们把这个声明语句往上面的形式套，则T是const int，类型限定词表是const volatile, D是a。  
现在我们去除掉指针符，即去掉\*和const volatile，剩下的是
```c
const int a；
```
在const int a;这个声明中，a的类型是const int，因此在const int * const volatile a;中，a是一个const volatile的，且指向const int类型的指针。

通过这种方法我们可以区分const int \*a;和int \* const a;  
前者我们去掉指针符后是const int a;则前者是指向const int的指针，因此a的指向内容不能改变。  
而后者中去掉指针符后是int a;则后者是一个const的，且指向int的指针，因此a是一个常量，我们不能改变a指向的地址

### 数组的语义 ###

和指数组相关的产生式是这一句：

直接声明符 := 直接声明符[常量表达式];

C Standard 6.7.5.2中关于数组的语义：
>If, in the declaration ‘‘T D1’’, D1 has one of the forms:  
>&emsp;D[ constant-expression(opt) ]  
>and the type specified for ident in the declaration ‘‘T D’’ is ‘‘derived-declarator-type-list T ’’, then the type specified for ident is ‘‘derived-declarator-type-list array of T ’’.

类似于分析指针要先去掉指针符，我们要分析数组类型的语义的话，要先去掉方括号，看去掉方括号后的类型，例如：

```c
int *a[6];
```
去掉[6]之后，a的类型是int\*，因此int \*a[6]中，a是int*的数组，且长度为6

C Standard中规定，若T D;中标识符的类型为T1，则T D[xxx];中标识符的类型为T1[xxx]（ 注意T1不一定就是T ）

在上例中我们的实际上是把a先和[6]结合，然后再和\*结合的。这也就是说数组方括号的优先级是高于指针的星号的，这一点可以从文法的产生式中得出。但如果我们想让a先和星号结合怎么办呢？答案是可以加括号，例如：
```c
int (*a)[6];
```
当加了括号之后，我们就要先分析指针的作用了，分析上例，我们先去掉指针符后是  
int (a)[6]，则这里是int的数组且长度为6，因此a是指向int[6]的指针。（ 至于分析int (a)[6]是先去掉[6]后剩下int (a)，然后根据前面我们知道int (a)和int a等价 ）

### 函数的语义 ###

函数的语法如下：

直接声明符 :=  
&emsp;&emsp;直接声明符(形参表) |  
&emsp;&emsp;直接声明符(标识符表(opt))

(上面第一个是函数声明，第二个是函数调用)

C Standard 6.7.5.3中关于函数的语义：
>If, in the declaration ‘‘T D1’’, D1 has the form  
>&emsp;&emsp;D( parameter-type-list )  
>or  
>&emsp;&emsp;D( identifier-list opt )  
>and the type specified for ident in the declaration ‘‘T D’’ is ‘‘derived-declarator-type-list T ’’, then the type specified for ident is ‘‘derived-declarator-type-list function returning T ’’.

同样，和指针和数组一样，分析函数的时候我们也是要先去掉一部分，分析函数的时候我们要去掉函数名后边的那个圆括号。

C Standard中规定：若T D;中标识符的类型是T1，则T D(xxxx);中标识符的类型是“返回T1的函数”（ 注意T1不一定就是T ）

例如：
```c
int *fun(int x, int y);
```
去掉圆括号是 int \*fun;在这个声明中fun的类型是int\*。因此int \*fun(int x, int y);中fun的类型是“返回int\*的函数”

### 语义分析的优先顺序 ###

由上面的分析可知，我们分析一个类型的方法就是先去掉一部分，然后分析剩下部分中的类型，指针要去掉指针符，数组要去掉方括号，函数要去掉圆括号。

但当我们把指针、数组、函数结合起来的时候就会产生一个问题：应当先分析哪一个的语义呢？例如：

```c
int (*apfi[3])(int *x, int *y);
```

从C Standard中所规定的声明符的语法中可以看出，数组和函数的优先级是高于指针符的，因为数组和函数在语法树中更深的地方，因此我们应当先分析数组和函数。但同时声明符的语法定义中规定了可以使用括号，当有括号的时候我们就应该先分析括号里面的。

总结一下就是：

1.先分析数组和函数  
2.如果有括号的话，先分析括号里的

在上面的例子中，(\*apfi[3])是被括号括起来的，因此我们应该先分析这个括号里的， 


括号里是\*apfi[3]，由于数组的优先级高于指针，因此这里应当先分析数组，去掉“[3]”，剩下int (\*apfi)(int \*x, int \*y);

由于(\*apfi)被括号括起来，因此要先分析指针，去掉“\*”，剩下int (apfi)(int \*x, int \*y);

这时的声明就符合函数的形式，去掉圆括号，剩下int(apfi);此时的apfi是int类型，则上一步中apfi是“返回int的函数”，再上一步中apfi是 “指向返回int的函数的函数指针”，再上一步，即一开始的apfi是函数指针的数组且长度为3

## typedef的用法 ##

当声明语句中的说明符中出现typedef的时候，这个声明语句的功能就是为这个声明语句中的类型命名一个别名，且别名的名称就是这个声明中的标识符的名字

例如：
```c
typedef int (*myNewType[3])(int *x, int *y);
```
这个typedef语句中命名了一个新的类型，新类型的名称是myNewType，myNewType所代表的类型是
```c
int (*a[3])(int *x, int *y);
```
这个声明语句中a的类型。

因此我们要使用typedef声明某个类型的时候只要三步：  
1.先写出一个声明这种类型的声明语句  
2.在前面加上typedef  
3.把标识符名改成新类型名

实际上typedef可以出现在说明符中的任何位置，不一定非要在开头，比如把上例写成：
```c
int typedef (*a[3])(int *x, int *y);
```
也是完全可以的

## 参考资料 ##
C Standard（ISO/IEC 9899:1999(E)）  
《C专家编程》  
《编译原理》




