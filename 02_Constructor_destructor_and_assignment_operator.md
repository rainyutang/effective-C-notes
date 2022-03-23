# 构造、析构、赋值运算

## 05、了解C++默认编写了哪些函数

声明一个空类后，编译器会自动为它声明一个拷贝构造函数、一个copy assignment操作符和一个析构函数。此外如果没有声明任何构造函数，编译器会默认声明一个default构造函数。所有这些函数都是public且inline的。

就好像如下代码：

```cpp
class Empty {
public:
    Empty(){};
    Empty(const Empty& rhs) {};
    ~Empty(){};
    Empty& operator= (const Empty& rhs){};
};
```

注意，编译器所产生的析构函数是non-virtual的，除非这个class的base-class有virtual析构函数。

如下是一个NamedObject template

```cpp
template<typename T>
class NamedObject {
public:
    NamedObject(const char* name, const T& value);
    NamedObject(const std::string& name, const T& value);
    ...

private:
    std::string nameValue;
    T objectValue;
};
```

如果自己声明了一个构造函数，编译器不会再声明defaul构造函数，这很重要，意味着编译器不会为你添加无实参构造函数。

NamedObject既没有copy构造函数，也没有copy assignment操作符，所以编译器会创建那些函数。

copy构造函数的用法：

```cpp
NamedObject<int> no1("Smallest Prime Number", 1);
NamedObject<int> no2(no1) //调用copy构造函数
```

编译器以```no1.nameValue```和```no1.objectValue```为初值初始化no2。nameValue类型为string，string有copy构造函数，所以会先调用string的copy构造函数并以no1.nameValue为实参，另一个成员是int，那是一个内置类型，no2.objectVale会copy no1.objectValue的每一个bits来完成初始化。

如果一个类内部含有reference成员或者const成员，编译器拒绝编译赋值操作，必须自己定义copy assignment操作符。

如果某个base class将copy assignment操作符声明为private，编译器拒绝为其derived classes生成copy assignment操作符。毕竟编译器为derived classes所生成的copy assignment操作符理应处理base class的部分，但是它们无法调用derived class无权调用的成员函数。

* **编译器可以自行为class创建default构造函数、copy构造函数、copy assignment操作符以及析构函数**

## 06、若不想使用编译器自动生成的函数，就应该明确拒绝

如果不想对象被拷贝，例如：

```cpp
HomeForSale h1, h2;
HomeForSale h3(h1); //不该通过编译
h1 - h2; //也不该通过编译
```

可以将copy构造函数和copy assignment声明为private：

```cpp
class HomoForSale {
public:
    ...

private:
    ...
    HomeForSale(const HomeForSale&);
    HomeForSale& operator=(const HomeForSale);
};
```

不需指定参数名称。当客户企图拷贝HomeForSale，编译器会阻挠，如果friend函数或者member函数之内这么做，连接器报错。

将连接期的错误转移至编译期是可能的，需要在一个专门阻止copying动作而设计的base class内，将copy函数和copy assignment声明为private。

```cpp
class Uncopyable{
protected:
    Uncopyable(){};
    ~Uncopyable(){};

private:
    Uncopyable(const Uncopyable);
    Uncopyable& operate=(const Uncopyable);
};

class HomeForSale:private Uncopyable{
    ...
};
```

* **为驳回编译器自动提供的功能，可将相应成员函数声明为private且不予实现。使用像Uncopyable这样的base class也是一种做法。**

## 07、为多态基类生成virtual函数

例如一个记录时间的base class：

```cpp
class TimeKeeper{
public:
    TimeKeeper();
    ~TimeKeeper();
    ...
};

class AtomicClock: public Timekeeper{...};
class WaterClock: public TimeKeeper{...};
class WristWatch: public TimeKeeper{...};
```

可以设计一个factory函数，返回一个指针指向计时对象：

```cpp
TimeKeeper* getTimeKeeper();
```

为了避免泄露，将factory函数返回的每一个对象用delete释放很重要：

```cpp
TimeKeeper* ptk = getTimeKeeper();
...
delete ptk;
```

getTimeKeeper()返回一个derived class对象，而指向它的指针则是base class的，目前base class有一个non-virtual析构函数。c++明确指出：**derived class对象经由base class对象指针释放，而该base class带有一个non-virtual的析构函数，其结果未定义，通常调用base class额析构函数，会导致derived class对象的一部分未被销毁。**

消除方法很简单，给base class一个virtual析构函数：

```cpp
class TimeKeeper{
public:
    TimeKeeper();
    virtual ~TimeKeeper();
    ...
};

TimeKeeper* ptk = getTimeKeeper();
delete ptk;
```

现在行为正确，析构函数在不同的的derived class内有不同的实现。如果class不含virtual函数，通常表示不意图被作为一个base class。同样，**当class不意图被作为base class， 不要令其析构函数为virtual。**

因为，如果想要实现virtual，对象必须包含某些信息，用来决定在运行时期哪一个virtual函数被调用，这份信息通常是一个vptr（virtual table pointer）。vptr指向一个由函数指针构成的数组，称为vtbl（virtual table）。每一个带有virtual函数的class都有一个相应的vtbl。对象调用某一个virtual函数，实际调用的函数取决于该对象vptr指向的vtbl。

许多人的心得是：**只有当class内含至少一个virtual函数，才为它声明virtual析构函数。**

有时候令class带有一个pure virtual函数可能更为便利。pure virtual函数导致abstract classes，也就是不能被实体化。如果希望建立一个抽象class，但是没有pure virtual函数，就可以将析构函数定义为pure virtual。

```cpp
class AWOV{
public:
    virtual ~AWOV() = 0;
}
```

这里有个窍门，你必须为这个pure virtual析构函数提供一份定义：

```cpp
AWOV::~AVOV(){}
```

因为，析构函数的运行方式是，最深层的derived的class的析构函数最先被调用，然后是每一个base class的析构函数被调用。编译器会在AWOV的derived class析构函数中创建一个对~AWOV()的调用，所以必须为这个函数提供一份定义，否则连接器会报错。

给base class一个virtual析构函数，这个规则只适用于polymorphic（带多态的）base class身上，这种base class的设计目的是使用base class的接口处理derived class对象。

并非所有base class的设计目的都是为了多态用途。例如标准的string和STL容器都不被设计作为base class使用。

* **polymorphic base class应该声明一个virtual析构函数，如果class带有任何virtual函数，他就应该拥有一个virtual析构函数。**

* **class的设计目的如果不是作为base class使用，或者不是为了具备polymorphically，就不应该声明virtual析构函数。



