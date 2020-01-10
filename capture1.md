# 1. Function Template

[TOC]

## 现在开始

### 定义一个模板

``` c++
//basics/max1.hpp 
template<typename T>
T max (T a, T b)
{
    // if b < a then yield a else yield b
    return b < a ? a : b;
}
```

这个函数模板定义了返回一个两变量最大值的函数，需要传入两个函数参数a和b。

> 仔细看就发现返回值并没有按照我们日常的代码书写逻辑`return a < b？b : a`。考虑下如下场景
>
> ```c++
> template <typename T> // T models TotallyOrdered
> inline
> void sort_2(T& x, T& y)
> {
>     if (y < x) swap(x, y);
>     assert(x == min(x, y));
>     assert(y == max(x, y));
> }
> ```
>
> 忽略我们未知的template知识，我们看到第二个断言按我们上述提到的代码会触发。

好，言归正传，分析下函数模板的语法知识。

1. `template< comma-separated-list-of-parameters >` 

   必须要有的是`template<>` 如上面的实例代码还有`typename `和`T`,`typename`关键字引入了类型参数，但这只是c++中通用的模板参数，其它的也可以，在这我们先按通用写法记住。类型参数是`T`，你可以用任何标识符做类型参数，`T`只是一个通用写法。

2. 如上例所示，你使用的任何类型`T`（如int, boolean,基本类型，类等）必须满足模板提供的操作。上例中使用了`<`操作，你传入的`T`类型必须满足此操作，才可以进行比较。另外，上述`max()`函数，类型`T`的值也必须是可以复制的才能返回。

### 使用

```c++
// basics/max1.cpp
#include "max1.hpp"
#include <iostream>
#include <string>

int main()
{
    int i = 42;
    std::cout << "max(7,i): " << ::max(7,i) << ’\n’;
    double f1 = 3.4;
    double f2 = -6.7;
    std::cout << "max(f1,f2): " << ::max(f1,f2) << ’\n’;
    std::string s1 = "mathematics";
    std::string s2 = "math";
    std::cout << "max(s1,s2): " << ::max(s1,s2) << ’\n’;
}
```

上述代码中，`max()`函数被调用三次，一次是`int,int`，一次`double, double`, 还有`string ,string`，模板会根据相应类型编译使用

```c++
int max (int a, int b)
{
    return b < a ? a : b;
}
double max (double a, double b)
{
    return b < a ? a : b;
}
```

`void`也是一个合法的模板类型参数，例如

```c++

template<typename T>
T foo(T*)
{
}
void* vp = nullptr;
foo(vp);
```

### 两阶段的变化

模板被“编译”在两个阶段：

1. 在定义时不实例化的情况下，将忽略模板参数来进行模板代码检查，这包括了
   - 语法错误是否发现
   - 使用了未知的、不依赖模板参数的命名（类型名，函数名）。
   - 检查不依赖模板参数的静态断言。
2. 在实例化时，再次检查模板代码以确保所有代码均有效。 也就是说，现在尤其要仔细检查所有依赖模板参数的部分。

看个例子

```c++
template<typename T>
void foo(T t)
{
    undeclared(); // first-phase compile-time error if undeclared() unknown
    undeclared(t); // second-phase compile-time error if undeclared(T) unknown
    static_assert(sizeof(int) > 10, // always fails if sizeof(int)<=10
    			"int too small");
    static_assert(sizeof(T) > 10, // fails if instantiated for T with size <=10
    			"T too small");
}
```

### 编译和链接

两阶段检查带来的一个重要的问题：当使用函数模板以触发其实例化的方式使用时，编译器（在某个时候）将需要查看该模板的定义。 当函数的声明足以编译其使用时，这将打破常规函数对常规编译和链接的区分。目前，让我们采用最简单的方法：在头文件中实现每个模板。

## 模板参数推导

当我们为某些参数调用函数模板（例如`max()`）时，模板参数由我们传递的参数确定。 如果我们将两个`int`传递给参数类型`T`，则C ++编译器必须得出结论，`T`必须为`int`。

但是，`T`可能也只是类型的一部分。例如，我们使用常量引用来声明`max()`,然后传递`int`, `T`再次被推导为`int`,因为函数参数匹配为`int const& `

```c++
template<typename T>
T max (T const& a, T const& b)
{
	return b < a ? a : b;
}
```

类型推导中的类型转换

**在类型推导中自动类型转换是被禁止的**

- 当通过引用来声明调用参数时，即使再微小的转换也不适用于类型推导。使用相同模板参数`T`声明的两个参数必须完全匹配。
- When declaring call parameters by value, only trivial conversions that decay are supported: Qualifications with `const` or `volatile` are ignored, references convert to the referenced type, and raw arrays or functions convert to the corresponding pointer type. For two arguments declared with the same template parameter `T` the decayed types must match.

例如：

```c++
template<typename T>
T max (T a, T b);
...
int const c = 42;
max(i, c); // OK: T is deduced as int
max(c, c); // OK: T is deduced as int
int& ir = i;
max(i, ir); // OK: T is deduced as int
int arr[4];
foo(&i, arr); // OK: T is deduced as int*
```

但这些是有误的：

```c++
max(4, 7.2); // ERROR: T can be deduced as int or double
std::string s;
foo("hello", s); // ERROR: T can be deduced as char const[6] or std::string
```

有三种方法避免错误：

1. 转换类型，让参数类型一致
2. 明确`T`类型，防止编译器类型推断
3. 指定参数有多个类型

```c++
max(static_cast<double>(4), 7.2); // OK
max<double>(4, 7.2); // OK
```

### 默认参数的类型推导

类型推导不适用于默认调用参数

```c++
template<typename T>
void f(T = "");
f(1); // OK: deduced T to be int, so that it calls f<int>(1)
f(); // ERROR: cannot deduce T

```

为了解决这个问题，你可以设置默认模板参数类型

```c++
template<typename T = std::string>
void f(T = "");
...
f(); // OK
```

## 多参数模板

举个例子

```c++
template<typename T1, typename T2>
T1 max (T1 a, T2 b)
{
	return b < a ? a : b;
}
...
auto m = ::max(4, 7.2); // OK, but type of first argument defines return type
```

如上例，定义了两个参数类型的`max()`函数，但是这样就带来一个问题。 如果您将其中一种参数类型用作返回类型，则无论调用者的意图如何，另一个参数的参数都可能转换为该类型。其返回值取决于调用参数的顺序。

C++提供不同方法来处理这个问题：

- 用第三个模板参数做返回值
- 让编译器处理返回类型
- 声明返回类型为两个参数类型的公共类型。

### 模板参数做返回类型

直接看例子，我们明确了返回类型

```c++
template<typename T>
T max (T a, T b);
...
::max<double>(4, 7.2); // instantiate T as double
```

如果模板和调用参数之间没有连接，并且无法确定模板参数，则必须在调用中显式指定模板参数。 例如，您可以引入第三个模板参数类型来定义函数模板的返回类型：

```c++
template<typename T1, typename T2, typename RT>
RT max (T1 a, T2 b);


template<typename T1, typename T2, typename RT>
RT max (T1 a, T2 b);
...
::max<int,double,double>(4, 7.2); // OK, but tedious


template<typename RT, typename T1, typename T2>
RT max (T1 a, T2 b);
...
::max<double>(4, 7.2) // OK: return type is double, T1 and T2 are deduced
```

如上代码的最后一种方法声明多个模板参数类型，明确返回类型，其余让模板进行推导。

### 推导返回类型

如果返回值依赖于模板参数，那么推断返回类型最行之有效的方法是让编译器找出，从C ++ 14开始，可以通过不声明任何返回类型（您仍然必须将返回类型声明为auto）来实现。

```c++
// basics/maxauto.hpp
template<typename T1, typename T2>
auto max (T1 a, T2 b)
{
	return b < a ? a : b;
}
```

实际上，对返回类型使用auto而不使用相应的尾随返回类型（`->`）表示必须从函数体内的return语句推导出实际的返回类型。 当然，必须可以从函数体中推断出返回类型。 因此，该代码必须可用，并且多个return语句必须匹配。

在C ++ 14之前，仅允许编译器通过或多或少地使函数的实现成为其声明的一部分来确定返回类型。 在C ++ 11中，我们可以从尾随返回类型语法允许我们使用调用参数这一事实中受益。 也就是说，我们可以声明返回类型是从什么运算符`？:`派生的：

```c++
template<typename T1, typename T2>
auto max (T1 a, T2 b) -> decltype(b<a?a:b)
{
	return b < a ? a : b;
}
```

在这，操作符`:?`的规则决定返回类型，这相当复杂，但是会产生直观的结果。（如果a和b有不同的类型，则返回他们的公共类型）。

```c++
template<typename T1, typename T2>
auto max (T1 a, T2 b) -> decltype(b<a?a:b);
```

上述代码是个声明，以便编译器可以使用`?:`操作符的规则去调用a和b去在编译的时候发现max()的返回类型。实现不一定必须匹配。 实际上，使用`true`作为运算符的条件`?:`在声明中就足够了。

```c++
template<typename T1, typename T2>
auto max (T1 a, T2 b) -> decltype(true?a:b);
```

但是，该定义都有一个明显的缺点：返回类型可能是引用类型，因为在某些情况下`T`可能是引用。 因此，您应该返回`T` `decay`的类型，如下所示：

```c++
#include <type_traits>
template<typename T1, typename T2>
auto max (T1 a, T2 b) -> typename std::decay<decltype(true?a:b)>::type
{
	return b < a ? a : b;
}
```

在这里，使用类型特征`std::decay <>`，它以成员类型返回结果类型。 它由`<type_trait>`中的标准库定义。 由于成员类型是一种类型，因此必须使用类型名来限定表达式才能访问它。 请注意，类型为auto的初始化始终会衰减。 当返回类型只是auto时，这也适用于返回值。 `auto`作为返回类型的行为与以下代码相同，其中`a`由`i`的衰减类型int声明：

```c++
int i = 42;
int const& ir = i; // ir refers to i
auto a = ir; // a is declared as new object of type int
```

### 返回类型做通用类型

从C ++ 11开始，C ++标准库提供了一种具体选择“更通用类型”的方法。`std::common_type<>::type`产生作为模板参数传递的两种（或多种）不同类型的“通用类型”。 例如：

```c++
#include <type_traits>
template<typename T1, typename T2>
std::common_type_t<T1,T2> max (T1 a, T2 b)
{
	return b < a ? a : b;
}
```

`std::common_type`是一个特征类型，定义在<type_traits>, 它产生一个结构，该结构具有用于所得类型的类型成员。 因此，其核心用法如下

`typename std::common_type::type // since C++11`

c++14对其简化为`std::common_type_t // equivalent since C++14`

## 默认模板参数

您还可以定义模板参数的默认值。这些值称为默认模板参数，可以与任何类型的模板一起使用。它们甚至可以引用以前的模板参数。 

例如，如果您想结合使用具有多个参数类型的能力来定义返回类型的方法（如前面部分所述），则可以为返回类型引入模板参数`RT`，并使用两者的通用类型 默认参数。 同样，我们有多种选择：

1. 我们可以直接使用操作符`?:`。 但是，因为我们必须应用` ?:`，所以在声明调用参数`a`和`b`之前，我们只能使用它们的类型。

   ```c++
   #include <type_traits>
   template<typename T1, typename T2,
   typename RT = std::decay_t<decltype(true ? T1() : T2())>>
   RT max (T1 a, T2 b)
   {
   	return b < a ? a : b;
   }
   ```

   再次记住`std::decay_t<>`的用法以确认没有引用被返回。还要注意，此实现要求我们能够为传递的类型调用默认构造函数。 还有另一种使用`std::declval`的解决方案，但是这使声明更加复杂。 

2. 我们也可以使用`std::common_type<>`类型特征去指定返回类型的默认值

   ```c++
   #include <type_traits>
   template<typename T1, typename T2,
   typename RT = std::common_type_t<T1,T2>>
   RT max (T1 a, T2 b)
   {
   	return b < a ? a : b;
   }
   ```

   再次注意，`std :: common_type <>`衰减，因此返回值不能成为引用.

在所有情况下，作为调用者，您现在都可以将默认值用作返回类型：`auto a = ::max(4, 7.2);`或在所有其他参数类型之后显式指定返回类型：`auto b = ::max<double,int,long double>(7.2, 4);`

但是，同样存在一个问题，我们必须指定三种类型才能仅指定返回类型。 相反，我们需要具有将返回类型作为第一个模板参数的能力，同时仍能够从参数类型中推断出它。 原则上，即使遵循不带默认参数的参数，也可以为引导函数模板参数设置默认参数：

```c++
template<typename RT = long, typename T1, typename T2>
RT max (T1 a, T2 b)
{
	return b < a ? a : b;
}
```

根据上述定义，我们可以调用：

```c++
int i;
long l;
...
max(i, l); // returns long (default argument of template parameter for return type)
max<int>(4, 42); // returns int as explicitly requested
```

但是，只有在模板参数存在“自然”默认值的情况下，这种方法才有意义。 在这里，我们需要模板参数的默认参数依赖于先前的模板参数。 原则上，这是可能的，但是该技术取决于类型特征，并使定义变得复杂。 由于所有这些原因，最好和最简单的解决方案是让编译器根据前面的介绍推导返回类型。

## 重载函数模板

像普通函数一样，函数模板也可以重载。 也就是说，您可以使用相同的函数名称使用不同的函数定义，以便在函数调用中使用该名称时，C ++编译器去决定要调用各种候选函数中的哪一个。 即使没有模板，该决定的规则也可能变得相当复杂。 在本节中，我们讨论涉及模板时的重载。 如果您不熟悉没有模板的重载的基本规则，请查看附录C，我们在其中提供了有关重载解决方案规则的相当详细的调查。 以下简短程序说明了如何重载功能模板：

```c++
// maximum of two int values:
int max (int a, int b)
{
	return b < a ? a : b;
}

// maximum of two values of any type:
template<typename T>
T max (T a, T b)
{
	return b < a ? a : b;
}
int main()
{
    ::max(7, 42); // calls the nontemplate for two ints
    ::max(7.0, 42.0); // calls max<double> (by argument deduction)
    ::max(’a’, ’b’); // calls max<char> (by argument deduction)
    ::max<>(7, 42); // calls max<int> (by argument deduction)
    ::max<double>(7, 42); // calls max<double> (no argument deduction)
    ::max(’a’, 42.7); // calls the nontemplate for two ints
}
```

一个有趣的例子是如何最大化的重载模板：

```c++
template<typename T1, typename T2>
auto max (T1 a, T2 b)
{
	return b < a ? a : b;
}

template<typename RT, typename T1, typename T2>
RT max (T1 a, T2 b)
{
	return b < a ? a : b;
}
```

当我们在调用时有如下例子：

- `auto a = ::max(4, 7.2)  // call the first template`
- `auto b = ::max<long double>(7.2, 4)  //call the second template`

但当我们调用`auto c = ::max<int>(4, 7.2);`  就会发生混淆错误。所以当有重载的函数模板时，你要自己确定你所要调用的函数模板。

一个有用的例子是更大化重载指针和普通C字符串模板。

```c++
#include <cstring>
#include <string>
// maximum of two values of any type:
template<typename T>
T max (T a, T b)
{
	return b < a ? a : b;
}
// maximum of two pointers:
template<typename T>
T* max (T* a, T* b)
{
	return *b < *a ? a : b;
}
// maximum of two C-strings:
char const* max (char const* a, char const* b)
{
	return std::strcmp(b,a) < 0 ? a : b;
}
int main ()
{
    int a = 7;
    int b = 42;
    auto m1 = ::max(a,b); // max() for two values of type int
    std::string s1 = "hey";
    std::string s2 = "you";
    auto m2 = ::max(s1,s2); // max() for two values of type std::string
    int* p1 = &b;
    int* p2 = &a;
    auto m3 = ::max(p1,p2); // max() for two pointers
    char const* x = "hello";
    char const* y = "world";
    auto m4 = ::max(x,y); // max() for two C-strings
}

```

**注意以上我们所有的`max()`函数重载都是传递值，请注意，在max（）的所有重载中，我们都按值传递参数。 通常，在重载功能模板时，最好不要进行过多更改。 您应该将更改限制为参数数量或显式指定模板参数。 否则，可能会发生意想不到的影响。 例如，如果您实现max（）模板以通过引用传递参数并将其重载为通过值传递的两个C字符串，则不能使用三参数版本来计算三个C字符串的最大值：**

```c++
#include <cstring>
// maximum of two values of any type (call-by-reference)
template<typename T>
T const& max (T const& a, T const& b)
{
	return b < a ? a : b;
}
// maximum of two C-strings (call-by-value)
char const* max (char const* a, char const* b)
{
	return std::strcmp(b,a) < 0 ? a : b;
}
// maximum of three values of any type (call-by-reference)
template<typename T>
T const& max (T const& a, T const& b, T const& c)
{
	return max (max(a,b), c); // error if max(a,b) uses call-by-value
}
int main ()
{
    auto m1 = ::max(7, 42, 68); // OK
    char const* s1 = "frederic";
    char const* s2 = "anica";
    char const* s3 = "lucas";
    auto m2 = ::max(s1, s2, s3); // run-time ERROR
}
```

上例是一个运行期错误，上例的错误就是发生在`max(max(a,b),c)`,因为里面的max返回了一个临时变量的引用，这个临时变量在return后就销毁了，所以外面的max就处理错误。

但注意到调用`::max(7, 42, 68)`时没有出现问题，因为这些`int`值在`main`中就创建了，所以不受影响。

综上在写重载函数模板的时候还是要考虑周全，确定所有的重载版本。但事实上，并不是所有的函数调用都是可见的。正如下例所示：

```c++
#include <iostream>
// maximum of two values of any type:
template<typename T>
T max (T a, T b)
{
    std::cout << "max<T>() \n";
    return b < a ? a : b;
}
// maximum of three values of any type:
template<typename T>
T max (T a, T b, T c)
{
    return max (max(a,b), c); // uses the template version even for ints
} // because the following declaration comes
// too late:
// maximum of two int values:
int max (int a, int b)
{
    std::cout << "max(int,int) \n";
    return b < a ? a : b;
}
int main()
{
    ::max(47,11,33); // OOPS: uses max<T>() instead of max(int,int)
}
```

## 一些思考

### 值传递还是引用传递

你可以会想到用引用不是可以避免很多的复制构造吗？为什么还要考虑值传递，下面这些情况就是对值传递有优势的：

- 语法简单
- 编译优化最好
- 移动语义让复制很简单
- 一些不需要拷贝或移动的场景

对于模板，特定的场景也有作用：

- 模板可能同时用于简单类型和复杂类型，因此为复杂类型选择方法可能会适得其反
- 作为调用者，您仍然经常可以决定使用`std::ref()`和`std::cref()`通过引用来传递参数
- 尽管传递字符串文字或原始数组始终会成为一个问题，但通常认为通过引用传递它们成为更大的问题

所有这些都将在第7章中详细讨论。在书中，我们通常会按值传递参数，除非某些功能只有在使用引用时才可行.

### 为什么不使用`inline`

通常，不必使用内联声明函数模板。与普通的非内联函数不同，我们可以在头文件中定义非内联函数模板，并将此头文件包含在多个转换单元中。

该规则的唯一例外是针对特定类型的模板的完全专业化，因此生成的代码不再通用（定义了所有模板参数）。

从严格的语言定义角度来看，内联仅表示函数的定义可以在程序中多次出现。但是，这也意味着向编译器提示应该对该函数的调用进行“内联扩展”：这样做可以在某些情况下产生更有效的代码，但是在许多其他情况下也可以使代码效率降低。

总而言之，编译器自己会去帮你，你不需要考虑这事。

### 为什么不使用`constexpr`

`constexpr`是c++11的新特性，是告诉编译器可以在编译期计算某些值，很多模板都可以用其在编译期计算，简单来说就是用编译期时间换运行期时间。

```c++
template<typename T1, typename T2>
constexpr auto max (T1 a, T2 b)
{
	return b < a ? a : b;
}
int a[::max(sizeof(char),1000u)];
std::array<std::string, ::max(sizeof(char),1000u)> arr;
```

我们会在后面来讨论，暂时我们先跳过。



### 总结

- 函数模板是为不用的模板参数定义了一群函数
- 当您根据模板参数将参数传递给函数参数时，函数模板会推导要为相应参数类型实例化的模板参数
- 你可以显式设置模板参数
- 你可以定义默认模板参数，可以是涉及到的之前的模板参数加没有的模板参数
- 可以重载模板函数
- 当重载模板函数时，你应该确定对任何的调用只允许有一个匹配。
- 重载功能模板时，请将更改限制为明确指定的模板参数
- 在调用函数模板之前，请确保编译器可以看到所有重载的函数模板版本