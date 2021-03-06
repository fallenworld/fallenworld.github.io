---
layout: post
title: java基础学习笔记
categories: Java
tags: Java
---

* content
{:toc}

因为感觉自己java语法基础实在是很拙计，于是决定最近补一补java，使用的书籍是《java核心技术卷一 基础知识》和《Java编程思想》。下面是我学习的过程中一些之前不知道或者掌握不太扎实的Java知识点，记录在这里，做一个知识记录。

## 概论

- 编译时一个类会编译成一个.class，而非一个文件编译为一个.class
- javac编译，java解释，javah头文件生成， jdb调试，javadoc文档生成，javap反汇编
- jar文件实质就是zip文件，用文件夹目录来表示包名

## 基础语法

#### 数据类型

 - 0b1101\_0010\_1101二进制常量(Java7新增)，下划线用来增加可读性
 - java数据类型所占据空间大小和CPU位数无关，int永远32位，long永远64位
 - 浮点数判断溢出/Not a number(0/0, 负数开方)：Double.isNaN(x)
 - 不像C/C++，不能用0/非0代表布尔值true/false
 - 高精度整数BigInteger，高精度浮点数BigDecimal
 - 14.5e6表示十进制科学计数法：1.45*10^7

#### 运算符

 - 移位运算符只能用来处理整数类型
 - 对char,byte,short移位会先转为int再移位，并且最终返回int
 - \>\>\>0逻辑右移，\>\>算术右移
 - 逻辑运算会短路运算

#### 控制流程

- 不像C/C++，不能在子块中定义和父块同名变量
- 没有goto，但是可以用带标签的break（可在任何块中使用）
- for循环的初始化和步进部分可以用逗号操作符（执行多个语句，定义多个变量）

#### 字符串

- char是2字节的Unicode字符，String是char序列
- String中可能得用多个char来代表一个字符
- charAt返回的是char，而不一定是字符
- length():char(代码单元)的数量，实际字符(代码点)数量:codePointCount(0, String.length())
- compareTo比较字典顺序，indexOf查找子串，trim删除头尾空格，replace替换
- new String(“x”)一定会创建新的字符串对象，无论”x”是否在字符串常量池中

#### 数组

- int[] a = new int[100];
- int[] a = {0, 1, 2, 3};  //相当于new int[4]
- new int[] {0, 1, 2, 3} 匿名数组
- 数组长度可以为0，和null不同，可以用于函数在异常情况下返回
- Arrays.toString转为字符串，Arrays.sort快排，Arrays.copyOf拷贝数组，Arrays.fill元素填充，Arrays.binarySearch二分查，Arrays.equals判断相等

## 面向对象
#### 对象概论

- 所有对象变量都是引用
- Java虚拟机会自己决定是否把函数变为内联函数
- 和C++一样，成员函数可以访问除了自身以外的对象的私有属性
- 可变参数类型相当于数组，调用函数时会自动new一个数组传给实参
- final变量只能进行一次赋值（不一定要初始化）

#### 初始化和释放

- 和C++一样，如果没定义构造器，则会默认有一个无参构造器；但如果定义了构造器，就不会提供默认无参构造器
- 在构造器中调用另一个构造器：this(...)，只能调用一个且要位于开始处
- 构造对象时，先执行类内初始化，后执行构造函数
- 类static和非static成员变量都会默认初始化，数字和字符初始化为0，boolean初始化为false，引用初始化为null。注意:局部变量不会默认初始化
- 非static的final成员必须初始化(类内/构造函数中)
- 初始化块：类中写语句块来初始化，初始化块在构造函数之前调用。静态初始化块：static
- finalize方会在对象被垃圾回收之前调用，但注意对象可能不被垃圾回收

#### 访问控制

- public类可以被其他包使用，非public类只能被同一个包使用
- 成员不指定则默认为包可见，注意：包可见与public不同
- 不像C++，protected成员可以被本包和子类访问

#### 继承

- 和C++一样，子类不能访问父类的私有成员
- super用于访问父类的成员（和C++的用父类类名不一样）
- 和C++一样，子类构造函数如果没调用父类构造函数，则会自动调用父类的无参数构造函数
- 子类覆盖父类的方法时，返回值类型可以更改（因为返回值类型不属于方法签名）
- 子类覆盖父类的方法时，访问权限可见性不能低于父类的
- 动态绑定的原理是每一个类有一个方法表（类似虚函数表），调用方法时检索方法表
- final类：不允许继承的类，final方法：子类不能覆盖的方法
- 强制类型转换可以把父类转为子类，转换失败则抛出ClassCast异常（类似dynamic_cast）
- 可用instanceof判断某个变量是否是某个类型的类，返回boolean类型，且null返回false
- abstract类：abstract类不能实例化。abstract类可以包含具体成员和abstract方法，abstract类的子类如果没有实现所有的abstract方法，则这个子类也是abstract类

#### 接口

- 接口中的所有方法自动地为public
- 接口中的所有变量自动地为public static final
- 接口可以继承接口

#### 内部类

- 内部类持有外部类的一个引用：OuterClass.this（编译器给内部类增加一个this%0域）
- 编译器给内部类默认添加InnerClass(OutterClass outer)构造函数，给其他构造函数添加一个参数
- 编译器把内部类翻译成名为OuterClass$InnerClass的常规类，虚拟机不知道内部类
- 编译器给外部类添加access&0使得内部类能够访问外部类的私有数据
- 在外部构造非静态内部类：outer.new InnerClass();
- 函数中可以定义局部内部类，局部内部类可以引用final局部变量
- 编译器给局部内部类的构造函数添加引用的final变量为参数，并添加一个final成员
- 匿名内部类中不能定义构造函数
- static内部类与C++中嵌套类类似
- 接口中的内部类自动成为static public

#### Object类

- 数组也是Object的子类
-equals：是否内容相等，==：是否引用相同，Object类的equals：检测引用是否相同。考虑二者可能都为null要用Object.equals(a,b)，建议覆盖equals时满足自反传递对称一致
- hashCode：哈希码，Object类的hashCode：对象的内存地址。Object.hash(...)：组合多个哈希码。Object.hashCode(x)：若为null则返回0
- toString：+、 print时会自动调用toString，Object类的toString：类名@hashCode
- clone：对象拷贝，Object类的clone：浅拷贝，并且为protected。

#### 对象包装器

- 有Integer，Double，Void，Boolean，Character ...... 等等
- 一旦构造了对象包装器之后，就不可以再修改其值
- String构造Integer：Integer.valueOf(s)，String转int：Integer.parseInt(s)，Integer转String：i.toString()
- 包装器在运算中会自动转为基本类型（拆箱），基本类型也可自动构造出包装器（拆箱）

#### 枚举

- 定义枚举实质上是定义了个类，并且这个类继承自Enum类，枚举值即为类的实例变量
- toString返回枚举值的名字字符串
- valueOf根据名字字符串返回枚举值的引用
- EnumClass.values返回枚举数组，ordinal：枚举值在枚举数组中的下标

#### 反射

- 获取Class：obj.getClass()，Class.forName(String)，XXX.Class（Class类可以类比C++的type_info）
- 相同类型的变量的Class是公共的
- class.newInstance() 创建一个类实例，会调用默认构造函数，若无默认构造函数则抛出异常
- class.get(Declared)Fields/getMethods/getConstructors:获取成员变量/方法/构造器数组
- mothod/constructor.getParameterTypes:参数类型，method.getReturnType:返回值类型，getModifiers:返回修饰符数值

#### 代理

#### 异常

- Error：java内部错误，RuntimeException：空指针数组越界等，IOException：其它异常

(未完待续)
