# 让自己习惯C++

## 01.视c++为一个语言联邦

## 02.尽量以`const`、`enum`、`inline`代替`#define`

或者说以编译器替换预处理器

```C++
#define ASPECT 1.653
```

`ASPECT`可能从未被编译器看到，如果报错很可能报1.653而不是`ASPECT`

解决办法是以常量代替上面的宏：

```C++
const double Aspect
```

`Aspect`肯定会被编译器看到，并记入记号表内。同时，使用`const`可能会比`#define`产生更少的代码，因为`#define`可能会被预处理器盲目地替换为多份，而`const`不会出现这种情况。

当我们以常量替换`#define`，应考虑两种特殊的情况。

1. 定义常量指针，必须写`const`两次：

    ```C++
    const char* const Name = "Rain";
    ```

    一个更好的办法是使用String：

    ```C++
    const std::string Name("Rain");
    ```

2. class专属常量

    为了将常量的作用域限制在class内部，它必须是一个class成员，为确保此常量至多只有一份实体，它必须是一个`static`成员。

    ```c++
    class GamePlayer{
    private:
        static const int NumTurns = 6; //这是一个声明式
        int scores[NumTurns];
    };
    ```

    如果它是一个class专属常量且是static且为整数类型（int chat bool），只要不取地址，只需声明而无需定义，如果要取地址，需要在实现文件中而不是头文件中定义：
    变量定义：用于为变量分配存储空间，可为变量指定初始值，变量有且只有一个定义。

    ```cpp
    const int GamePlayer::NumTurns; //定义
    ```

    因为已经在声明时获得初值，定义时可以不再赋值。

    **无法用`#define`创建一个class专属常量**，这意味着`#define`无法提供任何封装性。

    旧式编译器也许不支持上述语法，它们不允许static成员在其声明式上获得初值。而且所谓的“in-class 初值设定”也只允许对整数常量进行，如果编译器不支持上述语法，可以将初值放在定义式：

    ```cpp
    class CostEstimate{
    private:
        static const double FudgeFactor; //static class 常量声明
        ... //位于头文件内
    };

    const double CostEstimate::FudgeFactor = 1.35; //位于实现文件内
    ```

    还有一个例外是当在class编译期间需要一个class常量值，万一编译器不允许“in-class初值设定”，可以使用“The enum hack”补偿做法。一个属于枚举类型的数值可权充ints被使用，于是可定义如下：

    ```cpp
    class GamePlayer{
    private:
        enum {NumTurns = 5};

        int scores[NumTurns];
    };
    ```

    * enum hack的行为某方面说比较像`#define`而不是`const`，例如取一个const的地址是合法的，但取一个enum的地址是不合法的，取#define的地址也是不合法的。如果不想让别人用指针或引用指向你的某个整数常量，可以用enum实现这个约束。

    * 使用enum纯粹是为了实用主义，“enum hack”是模板元编程的基础技术。

另一个常见的`#define`误用情况是以它实现宏（macros）。宏看起来像函数，但不会导致函数调用带来的额外开销。例如：

```cpp
#define CALL_WITH_MAX(a, b) f((a) > (b) ? (a) : (b))
```

缺点太多。

可以使用template inline函数：

```cpp
template<typename T>
inline void callWithMax(const T& a, const T& b){
    f(a > b ? a : b);
}
```

不需要在函数本体中为参数加上括号，也不需要担心参数被求值多次。

我们对预处理器的需求并未完全消除，`#include`仍是必需品，`ifdef/infdif`仍扮演者控制编译的角色。

总结：

* **对于单纯常量，最好以`const`对象或enums替换`#define`。**

* **对于形似函数的宏（macros），最好用inline函数替换`#define`。**

## 02.尽可能使用`const`

如果关键字`const`出现在星号左边，说明被指物是常量，如果出现在星号右边，说明指针是常量。如果出现在两边，说明都是常量。

以下两种写法意义相同：

```cpp
void f1(const Widget* pw);

void f2(Widget const * pw);

//传入的参数都是一个指针，指向一个常量Widget对象。
```

以下是有关STL的，迭代器的作用相当于T*指针。声明迭代器为const像声明指针为const，表示这个迭代器不能指向其他内容，但它所指的内容是可以改动的。如果希望被迭代器所指的东西是不可改动的，需要`const_iterator`：

```cpp
std::vector<int> vec;

const std::vector<int>::iterator iter = vec.begin(); //iter作用类似 T* const

*iter = 10; //正确
++iter; //错误

std::vector<int>::const_iterator cIter = vec.begin(); //cIter类似const T*

*cIter = 10; //错误
++iter; //正确
```

`const`最具威力的做法是函数声明时的应用。在一个函数声明式内，`const`可以和函数返回值、各参数、函数自身（如果是成员函数）产生关联。

令函数返回是一个常量值，可以降低因客户错误造成的意外：

```cpp
class Rational {...};
const Rational operator* (const Rational& lhs, const Rational& rhs);

//这样声明可以避免以下这种情况：
Rational a,b,c;
(a * b) = c; //在a * b的结果上调用operator*
```

对于const参数，除非有需要改动的参数或local对象，否则请将它们声明为const。

### const成员函数

将const作用于成员函数，是为了确认该成员函数可以用于const对象身上。这类成员函数重要的两个理由：

* 使class接口容易被理解，可以知道哪个函数可以改动对象，哪个不行。

* 使操作const对象成为可能

有一个容易被忽视的事实：**两个成员函数如果只是常量性（constness）不同，可以被重载**

```cpp
class TextBlock {
public:
    ...
    const char& operator[] (std::size_t position) const //operator[] for const 对象
    {return text[positon];}

    char& operator[] (std::size_t position) //operator[] for non-const 对象
    {return text[position];}

private:
    std::string text;
}
```

真实程序中const对象大多用于passed by pointer-to-const 或者 passed reference-to-const，例如：

```cpp
void print(const TextBlock ctb)
{
    std::cout << ctb[0]; //调用const TextBlock::operator[]
    ...
}
```

需要注意，`operator[]`的返回类型是reference to char，而不是 char。如果返回的是char，下面的语句无法通过编译：

```cpp
TextBlock tb("Hello");
tb[0] = 'x';
```

有两种const成员函数的说法。一种是，如果class内部有一个指针。那么不改变指针的函数就可以是const成员函数。另一种是const成员函数需要指针指向的值也不改变。

一个使用方法：

```cpp
class CTextBlock {
public:
    ...
    std::size_t length() const;

private:
    char* pText;
    mutable std::size_t textLength; //用nutable表示这些变量总会被修改，即使是在const成员函数内。
    mutable bool lengthIsValid;
};

std::size_t CTextBlock::length() const
{
    if(!lengthIsValid){
        textLength = std::strlen(pText);
        lengthIsValid = true;
    }
    return textLength;
}
```

### 在const和non-const成员函数中避免重复

对于“bitwise-constness非我所欲”问题，`mutable`是一个解决办法，但不能解决所有问题。

假设`operator[]`执行很多功能，如果像上文编写两个重载函数，代码冗余。因此可以利用常量性转除。令non-const代码调用其const兄弟，这个过程需要一个转型动作。

```cpp
class TextBlock {
public:
    ...
    const char& operator[] (std::size_t position) const{
        ...
        return text[position];
    }

    char& operator[] (std::size_t position) {
        return
            const_cast<char&>( //将op[]返回值的const转除
                static_cast<const TextBlock&>(*this)[position] //为*this加上const并调用const op[]
            );
    }
private:
    std::string text;
}
```

**不要用const成员函数调用non-const成员函数**。

总结：

* 将某些东西声明为const可以帮助编译器侦测出错误用法。const可被施加于任何作用域内的对象、函数参数、函数返回类型、成员函数本体。

* 编译器强制实施bitwise-constness，但编写程序时应该使用“概念上的常量性”（conceptual-constness）。

* 当const和non-const成员函数有着实质等价的实现时，令non-const版本调用const版本可避免代码重复。

## 确定对象被使用前已经初始化

array不保证其内容被初始化，vector能够保证。

对于无任何成员类型的内置类型，都必须手动初始化

```cpp
int x = 0;
const char* text = "Hello";

double b;
std::cin >> b;
```

至于内置类型以外的任何其他东西，初始化落在构造函数身上。规则很简单：确保每一个构造函数都将对象的每一个成员初始化。

注意：**重要的是不要混淆赋值与初始化**

```cpp
class PhoneNumber { ... };
class ABEntry {
public:
    ABEntry(const std::string& address, const std::string& name, const std:list<PhoneNumber>& phones)
    {
        //这是赋值
        theAddress = address;
        theName = name; 
        thePhones = phones;
    }
private:
    ...
};
```

c++规定，对象的成员变量的初始化动作发生在进入构造函数本体之前，以上的做法，成员变量不是被初始化，而是被赋值。初始化时间发生的更早，发生于这些成员（例如`phones`）的defalt构造函数被自动调用之时（早于进入ABEntry构造函数本体的时间）。如果成员变量是内置类型，不保证在你所看到的那个复制动作的时间点之前获得初值。

构造函数的一个较佳写法如下，使用成员初值列（member initialization list）替换赋值动作：

```cpp
ABEntry::ABEntry(const std::string& address, const std::string& name, const std:list<PhoneNumber>& phones)
    :theName(name),
    theAddress(address),
    thePhones(phones)
{}
```

这个构造函数结果相同，但效率更高。第一版中，对于成员变量，会先调用成员变量的默认构造函数，再对其赋值，默认构造函数就浪费了。使用后者的好处在于，初值列中针对各个成员变量而设的实参，被拿去作为各成员变量的构造函数的实参。也就是说，`theName`以`name`为初值进行copy构造，`theAddress`以`address`为初值进行copy构造...。

对于大部分类型，单独调用copy构造函数比先调用默认构造函数再赋值高效的多，对于内置类型差别不大，但为了一致性最好也通过成员初值列来初始化。同样，甚至想要默认构造一个成员变量，也可以使用成员初值列，只要指定nothing作为初始化实参：

```cpp
ABEntry::ABEntry()
    :theName(),
    theAddress(),
    nums(0)
```

如果成员变量再成员初值列中没有被指定初值，编译器自动为该成员调用默认构造函数。

请立下一个规则：总是在成员初值列中列出所有成员变量，以免还得记得哪些可以无需初值。

如果class拥有多个构造函数，每个构造函数都有自己的初值列，多份成员初值列可能会导致重复，这时可以合理地在初值列中遗漏“赋值和初始化一样好”的成员变量，改用它们的赋值操作。可以将赋值操作单独写成某个函数（通常是private的），供所有构造函数调用。然而，成员初值列仍然更加可取。

c++有着固定的成员初始化次序，以其声明顺序被初始化。所以在初值列中列出成员函数是，最好以声明顺序列出。

接下来考虑不同编译单元之间non-local static对象的初始化顺序。

例如，在一个编译单元中：

```cpp
class FileSystem{ //来自你的程序库
public:
    ...
    std::size_t numDisk() const;
    ...
};
extern FileSystem tfs;
```

假设你的某些用户用上了tfs对象：

```cpp
class Directory{
public:
    Directory( prarms );
    ...
};

Directory::Directory( params )
{
    ...
    std::size_t disks = tfs.numDisks();
    ...
}
```

进一步假设，这些客户决定创建一个Directory对象：

```cpp
Directory tempDir( params );
```

现在，初始化顺序的重要性显现出来了，除非tfs在tempDir之前被初始化，否则tempDir的构造函数会调用尚未初始化的tfs。而tfs和tempDir又是不同的人在不同的时间用不同的源码文件建立起来的，它们是定义在不同编译单元内的non-loacl static对象。

可以使用一个设计解决这个问题。将每个non-local static对象搬到自己的专属函数内部（该对象在函数内被声明为static），这些函数返会一个reference指向它所含的对象：

```cpp
class FileSystem {...}; //同上

FileSystem& tfs()
{
    static FileSystem fs;
    return fs;
}
```

客户文件中：

```cpp
class Directory {...}; //同上
Directory::Directory( params )
{
    ...
    std::size_t disks = tfs().numDisks();
    ...
}
Directory& tempDir()
{
    static Directory td;
    return td;
}
```

这个方法的基础在于：c++保证，函数内的local static对象会在“函数被调用期间”“首次遇上该对象之定义”时被初始化。

总结：

* **为内置对象进行手工初始化，因为c++不保证初始化它们**
  
* **构造函数最好使用成员初值列，而不是在构造函数本体内使用赋值操作。初值列列出的成员变量，其排列次序应当和它们在class的声明次序相同。**

* **为免除“跨编译单元之初始化次序”问题，以loacl static对象替换non-local static对象**
