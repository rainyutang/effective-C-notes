# 资源管理

## 13、以对象管理资源

考虑如下例子：

```cpp
class Investment {...};
Investment* createInvestment();

void f()
{
    Investment* pInv = createInvestment();
    ...
    delete pInv;
}
```

当以下情况发生，delete不会被执行：

1. “...”内存在过早的return语句。

2. delete位于某循环内，而该循环由于goto或continue等过早退出。

3. “...”区域抛出异常。

为确保createInvestment返回的资源总是被释放，需要将资源放进对象内，当控制流离开f， 该对象的析构函数会自动释放那些资源，也就是说，把资源放进对象内，我们便可以依赖c++的析构函数自动调用机制确保资源被释放。

标准库提供的auto_ptr正是针对这种情况而设计的。auto_ptr是一个“类指针（point_like）对象”，也就是所谓的智能指针，其析构函数自动对其所指对象调用delete。

```cpp
void f()
{
    std::auto_ptr<Investment> pInv(createInvestment());
    ...
    //经由auto_ptr的析构函数自动删除pInv
}
```

这个例子示范了两个关键想法：

1. 获得资源后立即放进管理对象内

2. 管理对象运用析构函数确保资源被释放

由于auto_ptr被销毁时会自动释放它所指之物，所以需要注意别让多个auto_ptr指向同一对象。否则，会导致对象被删除一次以上。auto_ptr有一个性质：通过copy构造函数或者copy assignment复制它们，它们会变成NULL。而复制所得的指针将获得唯一所有权。

STL要求容器发挥正常的复制行为，因此这些容器不能使用auto_ptr。

auto_ptr的替代方案是“引用计数型智能指针”，例如可以使用shared_ptr，其复制行为是正常的。

```cpp
void f() {
    ...
    std::tr1::shared_ptr<Investment> pInv(createInvestment());
    ...
}
//且可以复制：
void f() {
    ...
    std::tr1::shared_ptr<Investment> pInv1(createInvestment());
    std::tr1::shared_ptr<Investment> pInv2(pInv1);
    pInv2 = pInv1;
    ...
}
```

因此，它们可被用于STL容器以及其他auto_ptr不适用的语句上。

auto_ptr和tr1::shared_ptr在其析构函数内使用的是delete而不是delete{}, 因此不能在动态分批的array上使用，虽然这么做依然可以通过编译：

```cpp
std::auto_ptr<std::string> aps(new std::string[10]);
```

(要实现这样的功能可以使用boost::scoped_array和boost::shared_array classes)

也可以自己实现资源管理类，但是需要考虑更多细节，见14和15

最好的方法是对createInvestment()接口进行修改。

* **为防止资源泄露，请使用RAII对象，它们在构造函数中获得资源并在析构函数中释放。**

* **两个常用的RAII classes分别是tr1::shared_ptr和auto_ptr，前者通常是较好的选择，其copy行为比较直观，后者复制动作会使其指向null。**

## 03、在资源管理类中小心copying行为

我们可能需要建立自己的资源管理类，例如，假设使用C API函数处理类型为Mutex的互斥对象，有lock和unlock两个函数可用：

```cpp
void lock(Mutex* pm);
void unlock(Mutex* pm);
```

建立一个class用来管理机锁，这样的class遵守RAII守则，也就是“资源在构造期间获得，在析构期间释放”：

```cpp
class Lock {
public:
    explicit Lock(Mutex* pm) : mutexPtr(pm) {
        lock(mutexPtr);
    }
    ~Lock() {
        unlock(mutexPtr)
    }
private:
    Mutex* mutexPtr;
};
```

用法也符合RAII：

```cpp
Mutex m;
...
{
    Lock m1(&m)
    ... //在区块末尾自动解除互斥器锁定
}
```

但是当对象被复制，会发生什么？

```cpp
Lock ml1(&m);
Lock ml2(ml1);
```

大多数情况，有两种选择：

### 禁止复制

也就是将copying操作声明为private， （条款6）

```cpp
class Lock: private Uncopyable {
    ...
};
```

### 对底层资源祭出“引用计数法”

可以利用shared_ptr:

```cpp
class Lock {
public:
    explicit Lock(Mutex* pm) : mutexPtr(pm, unlock) //unlock作为删除器，引用为0时运行
    {
        lock(mutexPtr.get());
    }
private:
    std::tr1::shared_ptr<Mutex> mutexPtr;    
};
```

不在需要析构函数，默认析构函数会调用所有非static对象的析构函数，而mutexPtr的析构函数会在引用次数为0时自动调用unlock。

### 复制底部资源

也就是进行深度拷贝，这种情况下复制资源管理对象，同时复制其所包含的资源。

### 转移底部资源的拥有权

这种情况类似auto_ptr，资源的拥有权会从被复制物转移到目标物。

编译器可能会自动创建copy构造函数和copy assignment操作符，所以需要注意是否默认函数做了你想做的事情。

* **复制RAII对象必须一并复制它所管理的资源，资源的copying行为决定了RAII对象的copying行为。**

* **普遍的RAII class copying行为是：抑制copying、使用引用计数法。**

## 15、在资源管理类中提供对原始资源的访问

有两种方式可以实现这个目的，分别是显式方法和隐式方法，例如：

```c++
FontHandle getFont();
void releaseFont(FontHandle fh)
class Font {
public:
    explicit Font(FontHandle fh) : f(fh) {}
    ~Font() {releaseFont(f)}
private:
    FontHandle f;
};

//显式方法，类似于shared_ptr以及auto_ptr的实现
class Font {
public:
    ...
    FontHandle get() const {return f;} //显式转换函数
}；

//隐式方法，令Font提供隐式转换函数
class Font {
public:
    ...
    operator FontHandle() const //隐式转换函数
    {return f;}
    ...
};
```

* **APIs往往要求访问原始资源，所以每一个RAII class应该提供一个“取得一个其管理的资源”的方法。**

* **对原始资源的访问可能经由显式转换或隐式转换。一般而言，显式转换比较安全，隐式转换对客户比较方便。**

## 16、成对使用new和delete时要采取相同的形式

一下是错误的：

```cpp
std::string* stringArray = new std::string[100];
delete stringArray;
```

规则很简单，当调用new的时候使用[]，必须在调用delete时也使用[]。

当撰写的class中有一个指针指向动态分配的内存，同时提供了多个构造函数，这个规则就很重要，你要小心的在所有构造函数中使用相同形式的new。

这个规则对喜欢使用typedef的人也很重要。

```cpp
typedef std::string AddressLines[4];

std::string* pal = new AddressLines;

delete pal;    //错误
delete [] pal; //正确
```

所以，为了避免此类错误，最好不要对数组形式使用typedef，c++标准程序库含有vector、string等templetes，几乎可将对数组的使用需求将为0。例如，上例中可以使用vector\<string>。

* **如果new中使用[]，delete中也要使用[]，反之亦然。**

## 17、以独立语句将newed对象置入智能指针

有以下两个函数：

```cpp
int priority();
void processWidget(std::shared_ptr<Widget> pw, int priority);

processWidget(new Widget, priority()); //这种调用方法是错误的，因为shared_ptr不提供隐式转换

processWidget(std::shared_ptr<Widget>(new Widget), priority()); //这是正确的，但是可能泄露资源
```

在调用processWidget之前，编译器要做三件事：

1. 调用priority()

2. 执行 new Widget

3. 调用shared_ptr构造函数

这三个的执行顺序是不确定的，我们只知道第三步会在第二部之后执行。

因此，编译器执行的顺序可能是这样的：

1. 执行 new Widget

2. 调用priority()

3. 调用shared_ptr构造函数

现在，万一priority()的调用出现异常，此时new Widget产生的指针将会遗失，从而引发资源泄露。

避免这类问题的方法很简单：使用分离语句，分别写出(1)创建Widget，(2)将它置于智能指针内，然后再把那个智能指针传给processWidget：

```cpp
std::shared_ptr<Widget> pw(new Widget);
processWidget(pw, priority()); //这个调用不会造成泄露
```

* **以独立语句将newed对象存储于智能指针内，否则，一旦发生异常，可能导致难以察觉的资源泄露。**
