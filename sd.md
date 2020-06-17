# 深度探索C++对象模型

标签（空格分隔）： C++ 深度探索C++对象模型

---
[toc]

## 关于对象
这是本书的第一章，可能会有一些同学不习惯本书的写作风格或者翻译风格，但是还请坚持读下去。

关于这一章可能会感觉云里雾里，不知所云，要注意每一节的侧重点，不要关注偏僻的地方，比如这章探讨对象，就不要在其他的地方太过纠结，这本书很多解释是站在编译器角度来看的，要考虑这一层。

带着问题去读，找到答案并理解

### C和C++
问题一：C和C++的程序风格有什么样的不同？
> C是面向过程，C++是面向对象，从很多书里我们能看到这样的解释。
这个区别不是编程的区别，而是设计上的区别，所以你当然可以使用C++写面向过程的程序。

如何通俗易懂地举例说明「面向对象」和「面向过程」有什么区别？ - 知乎
https://www.zhihu.com/question/27468564/answer/101951302

从书中例子看来，C很精炼，而C++相比起来显得非常繁杂，继承、模板仿佛让一个简单的程序变得复杂起来，因为它做的就是抽象化。
很直观的想到，这么复杂，程序肯定有很大负担，这样有意义吗？
然而，这样的设计在没有virtual的时候布局成本并没有增加，也没有影响执行效率。原因在于，类中的成员函数只会诞生一个函数实例（非内联函数），也就是多个该类生成的对象调用成员函数时，访问的是同一个地址。
如果是内联函数，则一个对象上会产生一个函数实例。

负担情况virtual有两种情况
virtual function
virtual base class

### C++对象模式
类实例化就是对象，考虑对象，首先从来源class说起。
先来说说C++ class中的成员，
class data members: static and nonstatic
class member functions : static, nonstatic, virtual

这些成员在实例化时候，在机器中会被怎么样表现呢？也就是内存上是如何的呢？
#### 简单对象模型
该模型内存上类似指针数组，每个元素（指针）指向一个member（data or function）
优点：编译器设计复杂度低
缺点：空间、执行效率低
对象大小：指针大小 * members个数

#### 表格驱动对象模型
该模型把members的信息抽出来，放在一个data member table和一个member function table中，class object本身仅内含指向这两个表格的指针。
table看作数组，data table元素为成员本身，member function table元素为函数的指针，指向成员函数。

优点：objects有一致的表达
缺点：空间、执行效率低
对象大小：指针大小 * 2

#### C++对象模型
nonstatic data members 配置在objects内（堆、栈）
static data members 则被存放在objects之外（数据区）
static member function 和 nonstatic member function放在objects之外（代码区）
virtual functions 两个步骤：
1. 每个class 产生一堆指向virtual functions的指针，放在表格之中，这个表格称为虚表virtual table(vtbl)
2. 每一个class object安排一个指针，指向虚表，这个指针称为虚指针vptr

提问：
如果有继承，那么deviced object是什么样的呢？

### 对象的差异
C++程序设计模型直接支持三种程序设计范式
程序模型（procedural model）
抽象数据类型模型（abstract data type model ADT）
面向对象模型（object-oriented model）


### 指针的类型
指针的类型决定了编译器从内存地址读取的范围，这样就和C语言的类型转换对上了。
而非指针和引用类型，基类对象由派生类构造则会引起截断。

---

## 构造函数语意学

### 隐式类型转换
在运算过程中，编译器可能自主的将一种类型转化为另一种类型
举个栗子
```cpp
class A
{
public:
    A(int ) {}
    A(int , int ) {}
    A(const A &) {}
    operator int() {}
}；

int main()
{
    A a = 1;
    a + 5;
}

```
上面main函数两个语句都会进行隐式转换，隐式转换发生了什么呢？
第一个语句，
A a = 1;
右边类型为int，左边类型为A，这是赋值运算符，右边类型和左边不同，所以不会调用复制构造函数，此时编译器将会尝试调用构造单一参数的构造函数，表达式隐式的变为
A a = A(1);
右边调用构造函数，转化为A类型。
很明显的看出，这里是将单一参数的构造函数看作了一个类型转化的"运算符"，另外，不要想当然认为只有单一参数的会受影响，构造函数的默认实参也会让你栽跟头，这种情况下，构造函数的参数数量是可变的。
对于第二个语句，
a + 5;
运算符右边5为int类型，支持+运算符
而运算符左边a没有定义+运算符，这时候编译器会检查可以使用的类型转化运算符，当找到operator int()，编译器就明白，这就是想要的，于是将a转换为int类型。
提问：
如果没有operator int()，但是A重载了+运算符，会发生什么？

隐式转化有优点，减少很多复制的情况发生，但是也带来了一定的不确定性，如何解决呢？
上面两种情况分别是其他类型转成A类型和A类型转为其他类型
对于A类型转为其他类型，控制重载操作运算符即可实现；
对于其他类型转为A类型呢？直接的想法是控制构造函数（单一参数），C++提供了eplicit关键字，将其放在构造函数前，表明只能显式调用。

### Default Constructor 的构造操作
默认构造函数在需要的时候被编译器产生出来。关键点：需要，是被编译器需要，而不是程序需要。
对程序而言，构造函数希望进行实例化，而且对类中的成员变量初始化工作，然而，对于编译器而言，程序想做的和我编译器有什么关系，我编译器指向执行完这条，然后执行下一条。这个默认的构造函数，仅仅为了编译器通过，除了确实能构建一个实例（仅分配空间，没有初始化），没有其他的功能。因此被形容为trivial constructor.
这里trivial 是指没有对类中成员进行初始化
nontrivial是指对类中成员进行了初始化

在一些特殊情况下，implicit default constructor是nontrivial

#### 带有Default Constructor 的 Member Class Object
简而言之，就是该类有 带有默认构造函数的成员对象。
如果Class A内含一个或一个以上的成员类对象，那么class A的每一个构造函数必须调用每一个成员类对象的默认构造函数。
如果Class A没有用户自定义的构造函数，那么由编译器生成的默认构造函数会调用成员类对象的默认构造函数，此时该类对成员类对象完成了初始化，但对其他普通成员（int等）类型依然不初始化，这样默认构造函数也是nontrivial。
另外，假如用户自定义的构造函数中没有显式的调用成员对象的构造函数，编译器会扩张已经存在的构造函数，使得构造函数在执行前，先调用必要的默认构造函数（按声明顺序）

#### 带有Default constructor 的 Base Class
与上一条类似，派生类会调用基类的默认构造函数，这样，构造函数时nontrivial，如果用户没有显式的对基类进行构造，那么派生类的构造函数将被扩张调用，如果同时存在派生类有 带默认构造函数的成员对象，在base class constructor调用后，会调用哪些成员对象的构造函数，隐式的顺序。

#### 带有一个Virtual Function 的 Class
```
class Widget 
{
public:
    virtual void flip() = 0;
};
void flip(const Widget & widget) {widget.flip(); }

class Bell : public Widget;
class Whistle : public Widget;
void foo()
{
Bell b;
Whistle w;

flip(b);
flip(w);

}
```
构造函数将会对虚函数处理
1. 生成虚函数表
2. 类对象中，生成虚函数表的指针，指向虚函数表
3. 改写调用操作
```
widget.flip()
->
*widget.vptr[1] (&widget)
```
1是虚函数表的索引
&widget是指this指针

虚表的第一个元素vptr[0]存放指针，指向type_info for Widget

#### 带有一个Virtual Base Class 的 Class
虚继承的情况，运行时才确定基类内存位置，
如果要访问虚基类的成员，我们需要虚基指针，再进行偏移，
而不是在派生类中偏移寻找。
因此，和上面一条情况相似，构造函数要设置虚基指针指向虚基类对象，所以当然不是nontrival的。
void foo (const A *pa) {pa -> __vbcX->i = 1024;}
__vbcX表示编译器产生的指针，指向virtual base class X

### Copy Constructor的构造操作
三种情况会以一个object的内容作为另一个class object的初值
```
Class X {...};
X x;

//1.显式复制
X xx = x;
//2.作为参数传递
void foo (X x);
//3.作为返回值传回
X foo_bar();
```

class设计者提供了copy constructor
X::X(const X& x);
X::X(const X& x, int y = 0);

如果class设计者没有提供explicit copy constructor，会如何呢？
编译器会进行default memberwise initialization
按成员逐次初始化，而且会递归进行，这里的递归指的是如果类中有其他类成员变量，进行copy construct时会对该类递归进行default memberwise initialization

default constructor分为trival和nontrival，依据是是否进行了初始化
copy constructor分为trival和nontrival，依据则是是否展现了bitwise copy semantics（位逐次拷贝）

编译器有两种复制对象的方法：bitwise copy & default memberwise copy


 - bitwise copy并没有调用复制构造函数，换句话说，当class展现bitwise copy semantics时，编译器没有为我们生成copy constructor，这种方法可能为memcopy直接拷贝，复制出的对象和原来的对象完全相同。
 - default memberwise copy 是对每个成员逐次赋值初始化一样，对于内置类型直接初始化，对于类类型递归调用其默认复制构造函数来初始化。
思考bitwise copy和shallow copy（浅拷贝）的区别
浅拷贝 专门对于指针的复制而不是指针所指的内容，而bitwise copy包含了这种情况

那么，什么时候编译器回采用bitwise copy，在什么情况下合成默认复制构造函数（即采用default memberwise copy）？下面四种情况回采用后者，否则前者

1. 当类含有类对象成员，且这个成员有复制构造函数（无论是编译器生成的还是显式定义的）
2. 当类继承自一个基类，且这个基类有复制构造函数（无论是编译器生成的还是显式定义的）
3. 当类含有虚函数
4. 当类有虚基类
 
如何理解上述的情况呢？对于1和2，复制数据成员和基类，既然提供了复制构造函数，可以认为它采用的是default memberwise copy
对于3和4，当出现用子类对象初始化父类对象时，由于不是使用引用或者指针，会引起截断，当使用虚函数时，会导致炸毁，所以合成的构造函数会显式的设定对象的虚指针vptr指向虚表，而不是直接拷贝，虚基类也是同理，设定对象的虚基指针。


### 程序转化语意学(Program Transformation)
#### 显式的初始化操作(Explicit Initialization)
显式初始化操作程序转化有两个阶段，定义重写（初始化剥离），安插赋值构造函数的调用
```C++
class X;

X x0(paras);
X x1 = X(Paras);
X x2(x0);
X x3 = X0;
X x4 = X(x0);

//====>转化
//1. 重新定义
X x0;   //初始化被剥除，即没有占用内存，可看作声明
X x1;   //初始化被剥除，即没有占用内存，可看作声明
X x2;   //初始化被剥除，即没有占用内存，可看作声明
X x3;   //初始化被剥除，即没有占用内存，可看作声明
X x4;   //初始化被剥除，即没有占用内存，可看作声明

//安插构造函数调用
x0.X::X(paras);
x1.X::X(paras);

//安插复制构造函数调用
x2.X::X(x0);
x3.X::X(x0);
x4.X::X(x0);

```

#### 参数的初始化（Argument Initialization）
```
class X;
void foo(X x0);

//调用
X xx;
foo(xx);
```

调用到底做了什么呢？
C语言的告诉我们，传递的不是对象或变量，而是一个临时的值；那这是怎么进行的呢？
```
//C++伪码
//编译器先产生了一个临时对象
X _temp0;

//编译器进行复制构造
_temp0.X::X( xx );

//重新 改写调用操作，使用临时对象
foo(_temp0);
```

能看出遗留问题吗？此时foo()的声明依然是传值调用，会无限的迭代下去，那么应该怎么做呢？答案是转化声明 void foo(X x0) --> void foo(X &x0)，这样就不会无穷迭代了。

#### 返回值的初始化 (Return Value Initialization)
```C++
//已知下面的函数定义
X bar()
{
    X xx;
    //处理 xx
    return xx;
}
```
那么bar函数是如何从局部对象xx中拷贝出来呢？有个双阶段的转化过程
1. 提前加一个参数，类型是class object的引用
2. return时候进行复制构造

```
//函数转换
//C++伪码
void
bar (X &__result)
{
    X xx;
    
    //编译器产生的默认构造调用操作
    xx.X::X();
    //处理xx
    
    //编译器产生的复制构造调用操作
    __result.X::X(xx);
    return;
}
```

```C++
//现在编译器有一个调用的操作，如下
X xx = bar();
```

