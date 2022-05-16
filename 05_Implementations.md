# 实现

## 26、尽可能延后变量定义式的出现时间

只要你定义了一个变量而其具有构造函数或者析构函数，那么你就得承受其构造成本，并在变量离开其定义域时承担析构成本，尽管这个变量可能并未被使用。

**你不只应该延后变量的定义，直到非得使用变量的前一刻为止，甚至应该尝试延后这份定义直到能够给它初值实参为止。**

也就是将:

```c++
std::string encrypted;
encrypted = password;
```

修改为：

```c++
std::string encrypted(password)
```

这样做可以避免毫无意义的default构造过程

**那么循环怎么办呢？**

```cpp
//方法A：定义于循环外
Widget w;
for(int i = 0; i < n; i++) {
    w = ...;
}

//方法B：定义于循环内
for(int i = 0; i < n; i++) {
    Widget w = ...;
}
```

* 方法A的成本：1个构造函数 + 1个析构哈数 + n个赋值操作

* 方法B的成本：n个构造函数 + n个析构函数

如果class的赋值成本低于一组构造+ 析构函数，那么A可能比较好。否则B比较好。且做法A造成的
w的作用域更大。所以，除非你知道赋值成本比构造+析构成本更低，或者正处理代码中对效率高度敏感的部分，否则，使用B。

## 27、尽量少做转型动作

C++提供四种新式转型：

```cpp
const_cast<T>(expression);
dynamic_cast<T>(expression);
reinterpret_cast<T>(expression);
static_cast<T>(expression);
```

* ```const_cast```通常被用来将对象的常量性转除。它也是唯一有此能力的c++style转型操作符。

* ```dynamic_cast```主要用来执行“安全向下转型”，也就是用来决定某对象是否归属继承体系中的某个类型。它是唯一无法由旧式语法执行的动作，也是唯一可能耗费重大运行成本的转型动作。
* ```reinterpret_cast```意图执行低级转型，实际动作（及结果）可能取决于编译器，也就表示它不可移植。例如将一个pointer to int转型为int。这一类转型在低级代码以外很少见。本书只使用一次，那是在讨论如何针对原始内存写出一个调试用的分配器时，见条款50。
* ```static_cast```用来强迫隐式转换，例如将non-const对象转换为const对象，或将int转为double等等。它也可以用来执行上述多种转换的反向转换，例如将void*指针转为typed指针，将pointer-to-base转为pointe-to-derived。但它无法将const转为non-const----这个只有const_cast能办到。

新式转型比旧式更受欢迎，因为：

1. 更容易在代码中被辨别出来。
2. 转行动作的目标要尽可能窄，编译器更可能诊断出错误的运用。例如，将const去掉只能使用新式的const_cast，否则无法通过编译。

我唯一使用旧式转型的时机是要调用explicit构造函数将对象传递给一个函数时。例如：

```cpp
class Widget {
public:
    explicit Widget(int size);
    ...
};

void doSomeWork(const Widget& w);
doSomeWork(Widget(15)); //旧式
doSomeWork(static_cast<Widget>(15)); //新式
```

转型并不只是编译器将一种类型视为另一种类型。任何一个类型转换（不论是显式还是隐式），往往真的令编译器编译出运行期间执行的码。例如将int转换成double几乎肯定会产出一些代码。

需要注意如下的代码：

```cpp
class Base {...};
class Derived : public Base {...};

Derived d;
Base* pb = &d;
```

pb和&d两个指针并不相同，（sizeof(*p)和sizeof(d)就是不同的。），会有一个偏移量offset在运行期间施行于Derived*指针身上，尽管它们代表的地址相同，但实际是不同的。所以，**“由于知道对象如何布局”而设计的转型，在某一平台行得通，在其他平台不一定行得通。**

另一件事是，我们很容易写出似是而非的代码。如下代码看起来对，实际是错误的：

```cpp
class Window {
public:
    virtual void onResize(){...};
    ...
};

class SpecialWindow: public Window {
public:
    vuirtual void onResize() {
        static_cast<Window>(*this).onSize();
        /*首先调用基类的onResize，这样写是不行的，
        它会创建一个副本并调用该副本的onResize，
        也就是说如果函数修改了class的某些成员变量，
        这个修改并不会运用于当前class上。
        */

        ...//进行专属行为
    }
}
```

解决办法是这么写：

```cpp
class SpecialWindow: public Window {
public:
    virtual void onResize() {
        Window::onResize(); //调用Window::onResize()作用于*this
        ...;
    }
};
```

接下来是dynamic_cast，需要注意dynamic_cast的许多实现版本非常慢。

之所以需要dynamic_cast，是因为你想在一个derived class对象身上执行derived class操作函数，然而你手上只有一个指向base的pointer或reference。有两个一般性做法可以避免这个问题

1. 使用容器并在其中存储直接执行derived class对象的指针（通常是智能指针）：

```cpp
class Window {...};
class SpecicalWindow: public Window {
public:
    void blink();
    ...
};

//不要这样使用：
typedef std::vector<std::shared_ptr<Window>> VPW;
VPW winPtrs;
...
for(VPW::iterator iter = winPtrs.begin(); iter != winPtrs.end(); iter++)
    if(SpecialWindow* psw = dynamic_cast<SpecialWindow*>(iter->get()))
        psw->blink();

//而应该改为这样做：
typedef std::vector<std::shared_ptr<specialWindow>> VPSW;
VPSW winPtrs;
...
for(VPSW::iterator iter = winPtrs.begin(); iter != winPtrs.end(); iter++)
    (*iter)->blink(); //iter是迭代器，用*取值，也就是对应的智能指针。
```

2. 在base class内提供virtual函数做你相对各个Window派生类做的事，也就是将blink声明于base class内，并提供一份什么也不做的缺省代码。

绝对要避免的一件事是，所谓的连串（cascading） dynamic_casts。

优良的c++代码很少使用转型，但它又是不可避免地，我们应该尽可能隔离转型动作，通常是把它隐藏在某个函数内，函数的接口会保护调用者不受函数内部任何肮脏龌龊动作的影响。

总结：

* **如果可以，尽量避免转型，特别是在注重效率的代码中避免dynamic_cast。如果有个设计需要转型动作，试着发展无需转型的替代设计。**
* **如果转型是必要的，试着将它隐藏在某个函数背后。客户随后可以调用该函数，而不需要将转型放在他们自己的代码内。**
* **宁可使用c++-style（新式）转型，不要使用旧式转型。前者很容易辨识出来，而且也比较有着分门别类的职掌。**

## 28、避免handles指向对象内部成员

假设你的程序涉及矩形。每个矩形由其左上角和右下角表示。为了让Rectangle对象尽可能小，你可能会决定不把定义矩形的这些点放在Rectangle对象内，而是放在一个辅助的struct内再让rectangle指向它：

```cpp
class Point { //用来表述点
public:
    Point(int x, int y);
    ...
    void setX(int val);
    void setY(int val);
    ...
};

struct RectDate {
    Point ulhc; //upper left-hand corner
    Point lrhc; //lower right-hand corner
};

class Rectangle {
...
private:
    std::shared_ptr<RectDate> pData;
};
```

如果要返回左上角和右下角，你可能会这样写：

```cpp
class Rectangle {
public:
    ...
    Point& upperLeft() const {return pData->ulhc};
    Point& lowerRight() const {return pData->lrhc};
    ...
};
```

这可以通过编译，但确实错误的。因为客户可以通过这两个返回的reference更改内部数据！

```cpp
Point coord1;
Point coord2;
const Rectangle rec(coord1, coord2);

rec.upperLeft().setX(50); //rec其实应该是不可变的！
```

这带给我们两个教训。

第一，成员变量的的封装性最多只等于“返回其reference”的函数访问级别。虽然ulhc和lrhc都被声明为private，但他们实际上是public，因为public函数传出了它们的reference。

第二，如果const成员函数传出一个reference。后者所指对象与对象自身有关联，而它又被存储与对象之外，那么这个函数的调用者就可以修改那份数据。

如果返回的是指针或者迭代器，这种情况依然会发生，因为reference， 指针， 迭代器都属于handle（用来取某个对象）。而返回一个代表对象内部数据的handle，随之而来的便是降低对象封装性的风险，它也可能导致虽然调用const成员函数却造成对象的数据被更改。

不被公开使用的成员函数也属于对象内部的一部分，因此也不应该返回它们的handle。也就是不应该令成员函数返回一个指针指向访问级别较低的成员函数。否则，后者的访问级别就会提高到和前者一样高的程度。因为可以通过返回的handle调用它。

上述的解决方法是在返回类型前加const：

```cpp
class Rectangle {
public:
    ...
    const Point& upperLeft() const {return pData->ulhc};
    const Point& lowerRight() const {return pData->lrhc};
    ...
};
```

这样涂写权就是被禁止的了。

然而，这可能在其他场合带来问题，他可能导致dangling handles（空悬的号码牌）。这种handle所指的对象不存在，这种不存在的对象最常见的来源就是函数返回值。例如某个函数返回GUI对象的外框，外框采用矩形形式：

```cpp
class GUIObject {...};
const Rectangle boundingBox(const GUIObject& pbj);
```

用户可能这么使用这个函数：

```cpp
GUIObject* pgo;
...
const Point* pUpperLeft = &(boundingBox(*pgo).upperLeft());
```

这显然是错误地，boundingBox会生成一个临时的Rectangle对象，函数返回后这个对象就销毁了，而指针pUpperLeft依然指向这个销毁了的对象内的成员。

这就是为什么不要返回handle代表对象内部成员，一旦违反，handle可能比所指对象更长寿。

然而也有例外，比如operator[]就允许引用string或者vector的内部元素，其返回值就是指向容器内部数据的reference。然而这只是特例，不是常态。

* **避免返回handles（reference， 指针， 迭代器）指向对象内部。遵守这个条款可以增加封装性，帮助const成员函数的行为像个const，并将发生dangling handles的可能性降至最低。**

## 29、为“异常安全”而努力是值得的

假设一个class用来实现GUI。这个class希望用于多线程环境，所以它有一个互斥器（mutex）作为并发控制之用：

```cpp
class PrettyMenu {
public:
    ...
    void changeBackground(std::istream& imgSrc);
    ...
private:
    Mutex mutex;
    Image* bgImage;
    int imageChanges;
};

//下面是changeBackground的一个可能实现：

void PrettyMenu::changeBackground(std:istream& imgSrc) {
    lock(&mutex);
    delete bgImage;
    ++imageChanges;
    bgImage = new Image(imgSrc);
    unlock(&mutex);
}
```

“异常安全”有两个条件，这个函数没有满足任何一个：

* **不泄露任何资源**。上述代码没有做到这一点，一旦new Image(imgSrc)导致异常，对unlock的调用就不会执行，互斥器就不会解锁。
* **不允许数据败坏**。new Image(imgSrc)抛出异常，bgImage指向一个已经被删除的对象，imageChange也会被累加，但其实并没有新的图像被安装。

积极而资源泄露的问题很容易，定义一个资源管理类，在其析构函数中解除互斥锁：

```cpp
void PrettyMenu::changeBackground(std::istream& imgSrc)
{
    Lock ml(&mutex); //见条款14
    delete bgImage;
    ++imageChange;
    bgImage = new Image(imgSrc);
}
```

因此我们可以专注于解决数据的败坏

在这之前，需要了解异常安全提供的三个保证：

* **基本承诺**：如果异常被抛出，程序内的任何事物仍然保持在有效的状态下。没有任何对象或数据结构会因此被破环，所有对象都处于一种内部前后一致的状态。
* **强烈保证**： 如果异常被抛出，程序状态不改变。调用这样的函数需要有这样的认知：如果函数成功，就是完全成功，如果函数失败，程序会回复到调用函数之前的状态。
* **不抛掷（nothrow）保证**：承诺绝不抛出异常，因为它们总能够完成它们预先承诺的功能。作用于内置类型（如指针，int等）身上的所有操作都提供nothrow保证。这是异常安全代码中一个必不可少的关键基础材料。

并不是函数带有空白的异常规范就代表其不会抛出异常，考虑以下函数：

```cpp
int doSomething() throw();
```

并不是说它绝不会抛出异常，而是说如果soSomething抛出异常，将会是严重错误，会有意想不到的函数被调用。实际上它没有提供任何异常安全性保证，这些保证是函数实现决定的，而不是声明的规范。

异常安全代码必须提供上述三种保证之一。否则，它就不具备异常安全性。我们的抉择是我们所写的函数应该提供哪一种保证。

实际上我们很难实现nothrow函数。任何使用动态内存的东西（例如STL）如果无法找到足够的内存，都会抛出bad_alloc异常（条款49）。

下面我们解决changeBackground的资源泄露问题：

```cpp
class PrettyMenu {
    ...
    std::shared_ptr<Image> bgImage;
    ...
};

void PrettyMenu::changeBackground(std::istream& imgSrc)
{
    Lock ml(&mutex);
    bgImage.reset(new Image(imgSrc));
    //注意：只有new Image(imgSrc)成功执行后，才会执行reset。
    //这种以对象管理资源的方式也再次缩短了函数的长度
    ++imageChanges;
}
```

然而这依然存在一些问题，如果Image构造函数出现异常，有可能输入流的读取记号已被移走，而这样的搬移对程序的其余部分是可见的，所以，它只提供基本的异常安全保证。这个可以解决，我们先不考虑这个问题。

有一个一般化的策略很容易实现强烈保证，即copy and swap。原则很简单，首先为需要修改的数据创建一份副本，在这个副本上进行修改，之后在一个不抛出异常的操作中置换(swap)。

```cpp
struct PMImpl {
    std::shared_ptr<Image> bgImage;
    int imageChanges;
};

class PrettyMenu {
    ...
private:
    Mutex mutex;
    std::shared_ptr<PMImpl> pImpl;
};

void PrettyMenu::changeBackground(std::istream& imgSrc)
{
    using std::swap;
    Lock ml(&mutux);
    std::shared_ptr<PMImpl> pNew(new PMImol(*pImpl));
    pNew->bgImage.reset(new Image(imgSrc));
    ++pNew->imageChanges;
    swap(pNew, pImpl);
}
```

有时提供强烈保证很困难，有时需要很多额外的开销，所以应该合理选择应该提供何种的异常保证。

当你撰写代码时，请仔细想想如何让他具备异常安全性。首先是“以对象管理资源”，防止资源泄露。然后是挑选三个安全保证中的某一个实施于你所写的每一个函数之上。你应该选择现实且可实施的条件下的最强等级，只有当你的函数调用了传统代码，才别无选择将它设为“无任何保证”。无论如何选择，将你的决定写成文档。函数的异常安全性保证是其可见接口的一部分，所以应该慎重选择，就像选择函数接口的其他任何部分一样。

* **异常安全函数即使发生异常也不会泄露资源或允许任何函数结构破坏。这样的函数区分三种可能的保证：基本型、强烈型、不抛任何异常型。**
* **“强烈保证”往往能够以copy and swap实现出来，但“强烈保证”并非对所有函数都可实现或具备现实意义。**
* **函数提供的“异常安全保证”通常最高只等于其所调用的各个函数提供的“异常安全保证”中的最弱者。**

## 30、透彻了解inlining的里里外外

