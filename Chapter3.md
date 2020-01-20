# Nontype Template Parameters

[TOC]

对于函数模板和类模板，模板参数不必是类型。 它们也可以是值。

## 非类型的类模板参数

 对比之前`Stack`的相同实现，你也可以实现一个固定大小数组元素的`Stack`。此方法的优点是可以避免由您或由标准容器执行的内存管理开销。 然而，为这种堆叠确定最佳尺寸可能是具有挑战性的。 您指定的大小越小，堆栈越可能充满。 指定的大小越大，不必要地保留内存的可能性就越大。 一个好的解决方案是让堆栈的用户将数组的大小指定为堆栈元素所需的最大大小。

为了做到这，就需要用户定义模板参数大小

```c++
#include <array>
#include <cassert>

template<typename T, std::size_t Maxsize>
class Stack {
private:
    std::array<T,Maxsize> elems; // elements
    std::size_t numElems; // current number of elements
public:
    Stack(); // constructor
    void push(T const& elem); // push element
    void pop(); // pop element
    T const& top() const; // return top element
    bool empty() const { // return whether the stack is empty
    	return numElems == 0;
	}
	std::size_t size() const { // return current number of elements
		return numElems;
	}
};
template<typename T, std::size_t Maxsize>
Stack<T,Maxsize>::Stack ()
: numElems(0) // start with no elements
{
	// nothing else to do
}
template<typename T, std::size_t Maxsize>
void Stack<T,Maxsize>::push (T const& elem)
{
    assert(numElems < Maxsize);
    elems[numElems] = elem; // append element
    ++numElems; // increment number of elements
}
template<typename T, std::size_t Maxsize>
void Stack<T,Maxsize>::pop ()
{
    assert(!elems.empty());
    --numElems; // decrement number of elements
}
template<typename T, std::size_t Maxsize>
T const& Stack<T,Maxsize>::top () const
{
    assert(!elems.empty());
    return elems[numElems-1]; // return last element
}

```

这个新的模板的第二个参数，是一个`int`类型。它具体确定了`Stack`内部数组元素大小

```c++
template<typename T, std::size_t Maxsize>
class Stack {
private:
    std::array<T,Maxsize> elems; // elements
    ...
};
```

而且，它还使用`push()`去检查`Stack`是否满了：

```c++
template<typename T, std::size_t Maxsize>
void Stack<T,Maxsize>::push (T const& elem)
{
    assert(numElems < Maxsize);
    elems[numElems] = elem; // append element
    ++numElems; // increment number of elements
}
```

使用这个模板。你必须却定义各个参数类型和最大数：

```c++
#include "stacknontype.hpp"
#include <iostream>
#include <string>
int main()
{
    Stack<int,20> int20Stack; // stack of up to 20 ints
    Stack<int,40> int40Stack; // stack of up to 40 ints
    Stack<std::string,40> stringStack; // stack of up to 40 strings
    // manipulate stack of up to 20 ints
    int20Stack.push(7);
    std::cout << int20Stack.top() << ’\n’;
    int20Stack.pop();
    // manipulate stack of up to 40 strings
    stringStack.push("hello");
    std::cout << stringStack.top() << ’\n’;
    stringStack.pop();
}
```

注意：每一个模板的实例化后它都拥有自己的类型。因此，`int20Stack`和`int40Stack`是两种不同的类型，并且没有定义它们之间的隐式或显式类型转换。 因此，不能代替一个使用另一个，也不能将一个分配给另一个。

再强调一次，模板默认参数也可以被明确指定：

```c++
template<typename T = int, std::size_t Maxsize = 100>
class Stack {
	...
};
```

但是，从良好设计的角度来看，这在此示例中可能不合适。 默认参数在直观上应该正确。 但是对于一般的堆栈类型，既不是类型`int`其最大也不是100。 因此，最好是程序员必须明确指定两个值，以便在声明期间始终记录这两个属性。

## 非类型的函数模板参数

和类一样，对函数也同样使用，可以定义一个非类型模板参数。举个例子，如下定义了一种确定值相加的函数模板：

```c++
template<int Val, typename T>
T addValue (T x)
{
	return x + Val;
}
```

如果将函数或操作用作参数，这种类型的函数可能会很有用。 例如，如果您使用C ++标准库，则可以传递此函数模板的实例化，以向集合的每个元素添加一个值：

```c++
std::transform (source.begin(), source.end(), // start and end of source
				dest.begin(), // start of destination
				addValue<5,int>); // operation
```

最后一个参数实例化函数模板`addValue <>（）`，以将`5`加到传递的`int`值上。 为原集合中的每个元素调用结果函数，同时将其转换为目标集合dest。

请注意，您必须为`addValue<>()`的模板参数`T`指定参数`int`。 推论仅适用于立即调用，并且`std::transform（）`需要一个完整的类型来推导其第四个参数的类型。 不支持仅替换/推导某些模板参数，而看不到适合的参数并推导其余参数。

再次提醒，你也可以从之前的参数推导确定模板参数，举个例子，从传递的非类型值去派生返回类型：

```c++
template<auto Val, typename T = decltype(Val)>
T foo();
```

或者去确定这个传递的值和传递的值有相同的类型

```c++
template<typename T, T Val = T{}>
T bar();
```

## 非类型模板参数的限制

请注意，非类型模板参数具有一些限制。 通常，它们只能是常量整数值（包括枚举），指向对象/函数/成员的指针，指向对象或函数的左值引用或`std::nullptr_t`（`nullptr`的类型）。

不允许将浮点数和类类型对象作为非类型模板参数:

```c++
template<double VAT> 		// ERROR: floating-point values are not
double process (double v) 	// allowed as template parameters
{
	return v * VAT;
}
template<std::string name> 	// ERROR: class-type objects are not
class MyClass { 			// allowed as template parameters
	...
};
```

将模板参数传递给指针或引用时，对象不能是字符串，临时对象或数据成员以及其他子对象。 由于在C ++ 17之前的每个C ++版本中放宽了这些限制，因此存在其他限制：

- C++11中，对象必须有外部链接
- C++17中，对象必须有内部和外部链接

因此，以下操作是不可能的：

```c++
template<char const* name>
class MyClass {
	...
};
MyClass<"hello"> x; // ERROR: string literal "hello" not allowed
```

但是这些有一些变通（依赖于C++版本）：

```c++
extern char const s03[] = "hi"; // external linkage
char const s11[] = "hi"; // internal linkage
int main()
{
    Message<s03> m03; // OK (all versions)
    Message<s11> m11; // OK since C++11
    static char const s17[] = "hi"; // no linkage
	Message<s17> m17; // OK since C++17
}
```

在这三种情况下，常量字符数组均由“ hello”初始化，并且此对象用作用`char const *`声明的模板参数。 如果对象具有外部链接（s03），则在所有C ++版本中都是有效的；在C ++ 11和C ++ 14中，如果对象具有内部链接（s11），则也是如此；从C ++ 17开始，如果对象根本没有链接 。

## 避免无效表达式

非类型模板参数可能是任何编译期表达式，举个例子

```c++
template<int I, bool B>
class C;
...
C<sizeof(int) + 4, sizeof(int)==4> c;
```

但是，如果`>`操作符被用在这个表达式，你必须给表达式带上括号。

```c++
C<42, sizeof(int) > 4> c; // ERROR: first > ends the template argument list
C<42, (sizeof(int) > 4)> c; // OK
```

## 模板参数类型—auto

从C++17，你可以定义一个非类型模板参数去匹配任意类型，它也允许非类型参数。基于这个特性，我们可以提供一个更通用的有固定大小的`Stack`类：

```c++
#include <array>
#include <cassert>
template<typename T, auto Maxsize>
class Stack {
public:
	using size_type = decltype(Maxsize);
private:
	std::array<T,Maxsize> elems; // elements
	size_type numElems; // current number of elements
public:
	Stack(); // constructor
	void push(T const& elem); // push element
	void pop(); // pop element
	T const& top() const; // return top element
	bool empty() const { // return whether the stack is empty
		return numElems == 0;
	}
	size_type size() const { // return current number of elements
    	return numElems;
    }
};


// constructor
template<typename T, auto Maxsize>
Stack<T,Maxsize>::Stack ()
	: numElems(0) // start with no elements
{
	// nothing else to do
}
template<typename T, auto Maxsize>
void Stack<T,Maxsize>::push (T const& elem)
{
    assert(numElems < Maxsize);
    elems[numElems] = elem; // append element
    ++numElems; // increment number of elements
}
template<typename T, auto Maxsize>
void Stack<T,Maxsize>::pop ()
{
	assert(!elems.empty());
	--numElems; // decrement number of elements
}
template<typename T, auto Maxsize>
T const& Stack<T,Maxsize>::top () const
{
	assert(!elems.empty());
	return elems[numElems-1]; // return last element
}
```

看一下定义：

```c++
template<typename T, auto Maxsize>
class Stack {
	...
};
```

通过`auto`这个占位符类型，你可以定义定义`Maxsize`时一个不确定类型的值。它可以是任何类型，它也被允许是一个非类型模板参数的类型。

在内部，你可以使用下面两个值和它的类型：

```c++
std::array<T,Maxsize> elems; // elements
using size_type = decltype(Maxsize);
```

例如，将其用作`size()`成员函数的返回类型：

```c++
size_type size() const { // return current number of elements
	return numElems;
}
```

从C++14起，你也可以使用`auto`来让编译器积极推导返回类型：

```c++
auto size() const { // return current number of elements
	return numElems;
}
```

使用此类声明，元素数量的类型由使用元素数量的类型定义，当使用`Stack`时：

```c++
#include <iostream>
#include <string>
#include "stackauto.hpp"
int main()
{
    Stack<int,20u> int20Stack; // stack of up to 20 ints
    Stack<std::string,40> stringStack; // stack of up to 40 strings
    // manipulate stack of up to 20 ints
    int20Stack.push(7);
    std::cout << int20Stack.top() << ’\n’;
    auto size1 = int20Stack.size();
    // manipulate stack of up to 40 strings
    stringStack.push("hello");
    std::cout << stringStack.top() << ’\n’;
    auto size2 = stringStack.size();
    if (!std::is_same_v<decltype(size1), decltype(size2)>) {
    	std::cout << "size types differ" << ’\n’;
    }
}
```

从`Stack<int,20u> int20Stack`可以看到这个内部类型是`unsigned int`,以为传递了`20u`。下面的`stringStack`也同理。

对于两个`stack`实例，`size()`也会有两个不同的类型，我们来看看他们的不同：

```c++
if (!std::is_same<decltype(size1), decltype(size2)>::value) {
	std::cout << "size types differ" << ’\n’;
}
```

这肯定会输出`size types differ`。

注意，对非类型模板参数类型的其他约束仍然有效。 特别是，前面讨论的关于非类型模板参数的可能类型的限制仍然适用。 例如：

```c++
Stack<int,3.14> sd; // ERROR: Floating-point nontype argument
```

并且，由于你还可能将字符串作为常量数组传递（因为C ++ 17甚至是本地静态声明），因此可能出现以下情况：

```c++
#include <iostream>
template<auto T> // take value of any possible nontype parameter (since C++17)
class Message {
    public:
    void print() {
    	std::cout << T << ’\n’;
    }
};
int main()
{
    Message<42> msg1;
    msg1.print(); // initialize with int 42 and print that value
    static char const s[] = "hello";
    Message<s> msg2; // initialize with char const[6] "hello"
    msg2.print(); // and print that value
}
```

还要注意，甚至`template <decltype(auto)N>`也是可能的，它允许实例化N作为参考：

```c++
template<decltype(auto) N>
class C {
...
};
int i;
C<(i)> x; // N is int&
```

## 总结

- 模板可以具有作为值而不是类型的模板参数。
- 你不能将浮点数或类类型的对象用作非类型模板参数的参数。 对于对字符串文字，临时对象和子对象的指针/引用，有一些限制。
- 使用`auto`可使模板具有非类型模板参数，这些参数是通用类型的值。