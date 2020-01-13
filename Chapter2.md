# 2. Class Tempaltes

与函数相似，类也可以使用一种或多种类型进行参数化。 容器类（用于管理某种类型的元素）是此功能的典型示例。 通过使用类模板，可以在元素类型仍处于打开状态时实现此类容器类。 在本章中，我们使用栈作为类模板的示例。

## 实现Stack模板类

```c++
#include <vector>
#include <cassert>
template<typename T>
class Stack {
private:
    std::vector<T> elems; // elements
public:
    void push(T const& elem); // push element
    void pop(); // pop element
    T const& top() const; // return top element
    bool empty() const { // return whether the stack is empty
    return elems.empty();
    }
};
template<typename T>
void Stack<T>::push (T const& elem)
{
	elems.push_back(elem); // append copy of passed elem
}
template<typename T>
void Stack<T>::pop ()
{
    assert(!elems.empty());
    elems.pop_back(); // remove last element
}
template<typename T>
T const& Stack<T>::top () const
{
    assert(!elems.empty());
    return elems.back(); // return copy of last element
}
```

很取巧的写法，用了`vector<T>`，我们也不用去考虑复制构造，拷贝构造，内存管理这些破事。不过这主要就是让我们来看类模板的写法。

### 定义一个类模板

常见有两种

```c++
template<typename T>
class Stack {
...
};
//Here again, the keyword class can be used instead of typename:
template<class T>
class Stack {
...
};

```

同样的，`T`代表各种类型，`push pop top`这些的操作类型都是`T`，这个类的类型就是`Stack<T>`，`T`是模板参数。每当在声明中使用此类的类型时，都必须使用`Stack <T>`，除非可以推断出模板参数。但是在类内部却不需要强制声明，如下例：

```c++
template<typename T>
class Stack {
    ...
    Stack (Stack const&); // copy constructor
    Stack& operator= (Stack const&); // assignment operator
    ...
};

//which is formally equivalent to:

template<typename T>
class Stack {
    ...
    Stack (Stack<T> const&); // copy constructor
    Stack<T>& operator= (Stack<T> const&); // assignment operator
    ...
};

```

但在类外，你就需要添加类型：

```c++
template<typename T>
bool operator== (Stack<T> const& lhs, Stack<T> const& rhs);
```

注意，在需要类名而不是类类型的地方，只能使用`Stack`。 当指定构造函数的名称（而不是其参数）和析构函数的名称时，尤其如此。 另请注意，与非模板类不同，您不能在函数或块作用域内声明或定义类模板。 通常，只能在全局/命名空间范围或内部类声明中定义模板

### 声明成员函数

要定义类模板的成员函数，必须指定它是模板，并且必须使用类模板的完整类型限定。 因此，类型为`Stack <T>`的成员函数`push()`的实现如下所示：

```c++
template<typename T>
void Stack<T>::push (T const& elem)
{
	elems.push_back(elem); // append copy of passed elem
}
```

其余成员函数如前面代码示例。

## 使用函数模板-Stack

```c++
#include "stack1.hpp"
#include <iostream>
#include <string>
int main()
{
    Stack<int> intStack; // stack of ints
    Stack<std::string> stringStack; // stack of strings
    // manipulate int stack
    intStack.push(7);
    std::cout << intStack.top() << ’\n’;
    // manipulate string stack
    stringStack.push("hello");
    std::cout << stringStack.top() << ’\n’;
    stringStack.pop();
}
```

实例化类模板的类型可以像其他任何类型一样使用。 您可以使用`const`或`volatile`对其进行限定，或者从中派生数组和引用类型。 您还可以将其用作typedef或使用typedef作为类型定义的一部分，或在构建另一个模板类型时将其用作类型参数。 例如：

```c++
void foo(Stack<int> const& s) // parameter s is int stack
{
    using IntStack = Stack<int>; // IntStack is another name for Stack<int>
    Stack<int> istack[10]; // istack is array of 10 int stacks
    IntStack istack2[10]; // istack2 is also an array of 10 int stacks (same type)
    ...
}

Stack<float*> floatPtrStack; // stack of float pointers
Stack<Stack<int>> intStackStack; // stack of stack of ints

Stack<Stack<int>> intStackStack; // ERROR before C++11
```

## 模板类的部分使用

在模板类中你不必对所有功能实现`T`的功能，没有实现的一些功能并不会影响你调用别的功能。

看一段代码, Stack中实现一个`printOn`,调用了`<<`操作符：

```c++
template<typename T>
class Stack {
    ...
    void printOn() (std::ostream& strm) const {
        for (T const& elem : elems) {
        	strm << elem << ’ ’; // call << for each element
        }
    }
};

Stack<std::pair<int,int>> ps; // note: std::pair<> has no operator<< defined
ps.push({4, 5}); // OK
ps.push({6, 7}); // OK
std::cout << ps.top().first << ’\n’; // OK
std::cout << ps.top().second << ’\n’; // OK

ps.printOn(std::cout); // ERROR: operator<< not supported for element type
```

但你仍然可以使用上面的函数调用，但是仅当您为此类调用`printOn`时，该代码才会产生错误，因为它无法实例化此特定元素类型的operator <<的调用：

## 友元

对于输出元素的最好实现是使用`operator <<`, 然后调用`printOn()`:

```c++
template<typename T>
class Stack {
    ...
    void printOn() (std::ostream& strm) const {
        ...
    }
    friend std::ostream& operator<< (std::ostream& strm,
    Stack<T> const& s) {
        s.printOn(strm);
        return strm;
    }
};
```

注意`operator <<`并不是一个函数模板，而是在需要时使用类模板实例化的普通函数。

但当我们试着去声明一个友元函数然后定义它之后，事情变得很复杂，事实上，我们有两个选择：

1. 我们可以隐式的声明一个新函数模板，它必须使用不同的模板参数，如`U`

   ```c++
   template<typename T>
   class Stack {
       ...
       template<typename U>
       friend std::ostream& operator<< (std::ostream&, Stack<U> const&);
   };
   
   ```

   在上例中，再次使用`T`或者跳过不使用模板参数声明均是无效的（要么内部T隐藏外部T，要么我们在命名空间范围内声明一个非模板函数）

2. 我们可以前项声明`Stack<T>`输出操作运算符作为一个模板，但是这意味着我们首先需要前置声明`Stack<T>`

   ```c++
   template<typename T>
   class Stack;
   template<typename T>
   std::ostream& operator<< (std::ostream&, Stack<T> const&);
   ```
   
   然后我们就可以用友元方式声明这个函数
   
   ```c++
   template<typename T>
   class Stack {
       ...
       friend std::ostream& operator<< <T> (std::ostream&, Stack<T> const&);
   };
   ```
   
   **注意这个`<T>`是紧邻`operator<<`这个函数名的。**因此，我们将非成员函数模板的特殊化声明为friend。 如果没有`<T>`，我们将声明一个新的非模板函数。
   
   无论如何，就像前面的`printOn`函数一样，没有定义它也没关系，只要调用它的时候才会抛出错误。
   
   ```c++
   Stack<std::pair<int,int>> ps; // std::pair<> has no operator<< defined
   ps.push({4, 5}); // OK
   ps.push({6, 7}); // OK
   std::cout << ps.top().first << ’\n’; // OK
   std::cout << ps.top().second << ’\n’; // OK
   std::cout << ps << ’\n’; // ERROR: operator<< not supported
   // for element type
   ```
   
   ## 类模板特化
   
   你可以为某些模板参数专门设置类模板。 与函数模板的重载类似，特化类模板使您可以优化某些类型的实现，或修复某些类型的行为不当的类模板的实例化。 但是，如果要专门化类模板，则还必须专门化所有成员函数。 尽管可以专门化类模板的单个成员函数，但是一旦完成后，就无法再专门化专门成员所属的整个类模板实例。
   要专门化类模板，您必须使用前导template <>和一个类模板专用的类型的规范。 类型用作模板参数，并且必须在类名之后直接指定：
   
   ```c++
   template<>
   class Stack<std::string> {
   ...
   };
   ```
   
   对于这些特化，任何成员函数的声明必须被定义为一个“普通”的成员函数，这些函数的类型`T`每次会被取代为专门的类型：
   
   ```c++
   void Stack<std::string>::push (std::string const& elem)
   {
   	elems.push_back(elem); // append copy of passed elem
   }
   ```
   
   来看一个完整的例子：
   
   ```c++
   #include "stack1.hpp"
   #include <deque>
   #include <string>
   #include <cassert>
   template<>
   class Stack<std::string> {
   private:
   	std::deque<std::string> elems; // elements
   public:
       void push(std::string const&); // push element
       void pop(); // pop element
       std::string const& top() const; // return top element
   	bool empty() const { // return whether the stack is empty
   		return elems.empty();
   	}
   };
   void Stack<std::string>::push (std::string const& elem)
   {
   	elems.push_back(elem); // append copy of passed elem
   }
   void Stack<std::string>::pop ()
   {
   	assert(!elems.empty());
   	elems.pop_back(); // remove last element
   }
   std::string const& Stack<std::string>::top () const
   {
       assert(!elems.empty());
       return elems.back(); // return copy of last element
   }
   ```
   
   这个例子中，`T`被特化为`std::string`, 特化使用引用去传递参数到push()， 对于特定类型这样更合适，对于复制引用移动我们后面再说。
   
   另一个区别是使用双端队列而不是向量来管理堆栈中的元素。 尽管这里没有特别的好处，但是它确实证明了特化的实现可能看起来与原始模板的实现非常不同。

## 局部特化

类模板时可以局部特化的，你可以按照你的需要去使用。例如，我们可以实现一个专门处理指针的Stack<>:

```c++
#include "stack1.hpp"
// partial specialization of class Stack<> for pointers:
template<typename T>
class Stack<T*> {
private:
	std::vector<T*> elems; // elements
public:
    void push(T*); // push element
    T* pop(); // pop element
    T* top() const; // return top element
    bool empty() const { // return whether the stack is empty
    return elems.empty();
	}
};

template<typename T>
void Stack<T*>::push (T* elem)
{
	elems.push_back(elem); // append copy of passed elem
}
template<typename T>
T* Stack<T*>::pop ()
{
    assert(!elems.empty());
    T* p = elems.back();
    elems.pop_back(); // remove last element
    return p; // and return it (unlike in the general case)
}
template<typename T>
T* Stack<T*>::top () const
{
    assert(!elems.empty());
    return elems.back(); // return copy of last element
}
```

这个类模板的意图还是挺清晰的，

```c++
template<typename T>
class Stack<T*> {
};
```

我们定义一个参数化模板，参数为`T`,但我们特化为指针-`T*`。对于这个特化的模板，pop接口就显著区别于以前，我们要对指针进行释放，所以必须返回。使用中：

```c++
Stack<int*> ptrStack; // stack of pointers (special implementation)
ptrStack.push(new int{42});
std::cout << *ptrStack.top() << ’\n’;
delete ptrStack.pop();
```

### 多参数的局部特化

直接看个例子：

```c++
template<typename T1, typename T2>
class MyClass {
	...
};
// partial specialization: both template parameters have same type
template<typename T>
class MyClass<T,T> {
	...
};
// partial specialization: second type is int
template<typename T>
class MyClass<T,int> {
	...
};
// partial specialization: both template parameters are pointer types
template<typename T1, typename T2>
class MyClass<T1*,T2*> {
	...
};

MyClass<int,float> mif; // uses MyClass<T1,T2>
MyClass<float,float> mff; // uses MyClass<T,T>
MyClass<float,int> mfi; // uses MyClass<T,int>
MyClass<int*,float*> mp; // uses MyClass<T1*,T2*>

MyClass<int,int> m; // ERROR: matches MyClass<T,T>
					// and MyClass<T,int>
MyClass<int*,int*> m; // ERROR: matches MyClass<T,T>
					  // and MyClass<T1*,T2*>
```

和第一章介绍函数模板出错问题一样，两个错误都是多个匹配混淆。解决第二个问题，可以给相同指针提供部分特化：

```c++
template<typename T>
class MyClass<T*,T*> {
	...
};
```

## 默认类模板参数

如函数模板一样，类模板也可以定义默认参数。

```c++
#include <vector>
#include <cassert>
template<typename T, typename Cont = std::vector<T>>
class Stack {
private:
	Cont elems; // elements
public:
    void push(T const& elem); // push element
    void pop(); // pop element
    T const& top() const; // return top element
    bool empty() const { // return whether the stack is empty
    	return elems.empty();
    }
};
template<typename T, typename Cont>
void Stack<T,Cont>::push (T const& elem)
{
	elems.push_back(elem); // append copy of passed elem
}
template<typename T, typename Cont>
void Stack<T,Cont>::pop ()
{
    assert(!elems.empty());
    elems.pop_back(); // remove last element
}
template<typename T, typename Cont>
T const& Stack<T,Cont>::top () const
{
    assert(!elems.empty());
    return elems.back(); // return copy of last element
}

```

注意看到我们现在有了两个模板参数，所以每次定义时候都用的是两个。你也可以和前面一样使用一个参数去实例化这个类模板，但它的默认元素容器是`std::vector<>`。你也可以特别使用别的容器，如`std::deque<>`:

```c++
#include "stack3.hpp"
#include <iostream>
#include <deque>
int main()
{
    // stack of ints:
    Stack<int> intStack;
    // stack of doubles using a std::deque<> to manage the elements
    Stack<double,std::deque<double>> dblStack;
    // manipulate int stack
    intStack.push(7);
    std::cout << intStack.top() << ’\n’;
    intStack.pop();
    // manipulate double stack
    dblStack.push(42.42);
    std::cout << dblStack.top() << ’\n’;
    dblStack.pop();
}
```

## 类型别名

通过类型别名更方便的使用模板

### Typedef 和 Using

类型别名使用`typedef`和`using`两关键字可以实现此功能

```c++
typedef Stack<int> IntStack; // typedef
void foo (IntStack const& s); // s is stack of ints
IntStack istack[10]; // istack is array of 10 stacks of ints


using IntStack = Stack<int>; // alias declaration
void foo (IntStack const& s); // s is stack of ints
IntStack istack[10]; // istack is array of 10 stacks of ints
```

### 别名模板

与`typedef`不同，`using`可以为模板提供别名，使用也如前面一样。**需要注意使用作用域。**

```c++
template<typename T>
using DequeStack = Stack<T, std::deque<T>>;
```

### 成员类型的别名模板

别名模板对类成员的类型简洁化特别有用，看下面例子：

```c++
struct C {
	typedef ... iterator;
	...
};
//or:
struct MyType {
    using iterator = ...;
    ...
};

//a definition such as
template<typename T>
using MyTypeIterator = typename MyType<T>::iterator;

//using
MyTypeIterator<int> pos;
```

`typename`关键字在这是必须的，因为这个成员是一个类型。

### 类型特性后缀

从c++14开始不需要加类型后缀`typename type`

```c++
std::add_const_t<T> // since C++14
typename std::add_const<T>::type // since C++11
    
namespace std {
	template<typename T> using add_const_t = typename add_const<T>::type;
}
```

## 类模板参数推导

在c++17以前，我们都需要传递所有参数到模板，除了默认参数模板。在C++17，这个约束被减弱，如果能完全推导出模板参数，就不必传递参数到模板。

举个例子，在所有之前的例子，你在没有具体化模板参数时就可以使用复制构造

```c++
Stack<int> intStack1; // stack of strings
Stack<int> intStack2 = intStack1; // OK in all versions
Stack intStack3 = intStack1; // OK since C++17
```

通过提供一个初始化一些参数的构造函数，你可以你可以推导出`Stack`的每个元素类型。举个例子，我们提供一个被单个元素初始化的`Stack`:

```c++
template<typename T>
class Stack {
private:
	std::vector<T> elems; // elements
public:
    Stack () = default;
    Stack (T const& elem) // initialize stack with one element
    	: elems({elem}) {
    }
    ...
};
```

现在允许你这样初始化:

`Stack intStack = 0;  // Stack<int> deduced since C++17`

通过这个初始化的`0`,这个模板参数类型`T`被推导为`int`,因此这个`Stack<int>`就初始化成功了。

需要遵循以下条件：

- 由于定义了int构造函数，因此您必须请求默认构造函数以其默认行为可用，因为默认构造函数仅在未定义其他构造函数的情况下才可用：

  `Stack() = default;`

- 参数elem传递给带有括号的elems，以使用elem作为唯一参数的初始化列表初始化vector elems

  `: elems({elem})`

  没有可以直接将单个参数作为初始元素的向量的构造函数

请注意，与函数模板不同，类模板参数不能仅部分推导（通过显式仅指定一些模板参数）。

### 字符串字面量的类模板参数推导

原则上，你甚至可以用字符串字面量来初始化`Stack`:

`Stack stringStack = "bottom"; // Stack deduced since C++17`

但这会带来很多麻烦：通常，当通过引用传递模板类型T的参数时，参数不会衰减(术语`decay`)，这是将原始数组类型转换为相应的原始指针类型的机制的术语。 这意味着我们真的初始化了一个

```c++
Stack<char const[7]>
```

而且这表示在任何使用`T`的地方使用`char const[7]`。举个例子，我们不能放一个不同大小的`string`进去，因为她有不同类型。

但是，当通过值传递模板类型T的参数时，参数类型会衰减(decays)，这是将原始数组类型转换为相应的原始指针类型的机制的术语。 即，构造函数的调用参数`T`被推导为`char const *`，从而整个类推导为`Stack <char const *>`。

因此，可能值得声明构造函数，以使参数通过值传递:

```c++
template<typename T>
class Stack {
private:
	std::vector<T> elems; // elements
public:
    Stack (T elem) // initialize stack with one element by value
    	: elems({elem}) { // to decay on class tmpl arg deduction
	}
	...
};
```

这样的话，以下的初始化才会成功：

```c++
Stack stringStack = "bottom"; // Stack<char const*> deduced since C++17
```

本例中我们最好是使用移动语义避免拷贝：

```c++
template<typename T>
class Stack {
private:
	std::vector<T> elems; // elements
public:
    Stack (T elem) // initialize stack with one element by value
    	: elems({std::move(elem)}) {
	}
	...
};
```

### 推导指南

除了声明构造函数要按值调用之外，还有一个不同的解决方案：因为在容器中处理原始指针是麻烦的源头，所以我们应该为容器类禁用自动推断原始字符指针的功能。
您可以定义特定的推导指南，以提供更多或修复现有的类模板参数推导。举个例子，你可以定义一个无论何时字符串字面量或者C字符串传递，`Stack`都能被初始化：

```c++
Stack(char const*) -> Stack<std::string>;
```

该指南必须出现在与类定义相同的作用域（命名空间）中。 通常，它遵循类定义。我们使用`->`来推导类型

现在声明为：

```c++
Stack stringStack{"bottom"}; // OK: Stack<std::string> deduced since C++17
                             // it deducts the stack be Stack<std::string>
Stack stringStack = "bottom"; // Stack<std::string> deduced, but still not valid
```

我们如下推导`std :: string`，以便实例化`Stack <std :: string>`：

```c++
class Stack {
private:
	std::vector<std::string> elems; // elements
public:
    Stack (std::string const& elem) // initialize stack with one element
    : elems({elem}) {
    }
    ...
};
```

但是，根据语言规则，您无法通过将字符串文字传递给需要`std :: string`的构造函数来复制初始化（使用`=`初始化）的对象。 因此，您必须按如下所示初始化堆栈

```c++
Stack stringStack{"bottom"}; // Stack<std::string> deduced and valid
```

**注意：**如有疑问，请使用类模板推导副本。在将`stringStack`声明为`Stack <std::string>`之后，以后初始化声明相同的类型（因此，调用复制构造函数），而不是通过字符串堆栈的元素初始化堆栈：

```c++
Stack stack2{stringStack}; // Stack<std::string> deduced
Stack stack3(stringStack); // Stack<std::string> deduced
Stack stack4 = {stringStack}; // Stack<std::string> deduced
```

## 模板聚合

聚合类也可以成为模板，例如：

```c++
template<typename T>
struct ValueWithComment {
    T value;
    std::string comment;
};
```

定义针对其所持有的`val`类型的参数化的聚合。 您可以像其他任何类模板一样声明对象，并且仍将其用作聚合：

```c++
ValueWithComment<int> vc;
vc.value = 42;
vc.comment = "initial value";
```

c++17你可以使用推导指向（deduction guide）来定义类模板：

```c++
ValueWithComment(char const*, char const*) -> ValueWithComment<std::string>;
ValueWithComment vc2 = {"hello", "initial value"};
```

没有推导指向（deduction guide），初始化将无法进行，因为`ValueWithComment`没有构造器去执行推导比对。

标准库`std::array<>`也是一个集合，参数化每一个类型和大小。C ++ 17标准库还为其定义了一个推导指向（deduction guide）

## 总结

- 类模板是使用一个或多个类型参数实现的类
- 要使用类模板，请将开放类型作为模板参数传递。 然后针对这些类型实例化（并编译）类模板。
- 对于类模板，仅实例化那些被调用的成员函数。
- 您可以针对某些类型专门设置类模板。
- 可以对类模板特化
- 从C ++ 17开始，可以从构造函数中自动推断出类模板参数。
- 你可以定义聚合类模板
- 如果声明由值调用，则模板类型的调用参数会衰减。
- 模板只能在全局/命名空间范围或类声明中声明和定义