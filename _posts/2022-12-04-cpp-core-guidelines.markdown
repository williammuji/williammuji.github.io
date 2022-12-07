---
layout: post
title:  "Cpp Core Guidelines"
date: 2022-12-04 14:00:00 +0800
categories: jekyll update
---

R:资源管理


## R:资源管理

资源一般需要获取和释放，比如内存，文件句柄，套接字，锁等。为什么需要释放的原因是，资源是有限的。如果不释放甚至延迟释放都可能造成问题。  
本节目的：如何不造成内存泄漏，以及避免持有资源时间过长。

- 资源管理规则摘要
	- [R.1:使用资源句柄和RAII来管理资源](#r1使用资源句柄和raii来管理资源)
	- [R.2:接口使用裸指针来代表单个对象](#r2接口使用裸指针来代表单个对象)
	- [R.3:T\*不拥有所有权](#r3t不拥有所有权)
	- [R.4:T&不拥有所有权](#r4t不拥有所有权)
	- [R.5:优先使用scoped对象，非必要不使用堆分配](#r5优先使用scoped对象非必要不使用堆分配)
	- [R.6:避免使用非const全局变量](#r6避免使用非const全局变量)
- 分配释放规则摘要
	- [R.10:避免使用malloc()和free()](#r10避免使用malloc和free)
	- [R.11:避免显式调用new和delete](#r11避免显式调用new和delete)
	- [R.12:资源的分配应立即丢给一个可管理对象](#r12资源的分配应立即丢给一个可管理对象)
	- [R.13:在一个表达式中最多执行一次显式资源分配](#r13在一个表达式中最多执行一次显式资源分配)
	- [R.14:避免使用[]，用span](#r14避免使用用span)
	- [R.15:重载allocation/deallocation要配套](#r15重载allocationdeallocation要配套)
- 智能指针规则摘要
	- [R.20:用unique\_ptr和shared\_ptr来表达所有权](#r20用unique_ptr和shared_ptr来表达所有权)
	- [R.21:优先使用unique\_ptr，除非要共享所有权才用shared\_ptr](#r21优先使用unique_ptr除非要共享所有权才用shared_ptr)
	- [R.22:使用make\_shared来获取shared\_ptr](#r22使用make_shared来获取shared_ptr)
	- [R.23:使用make\_unique来获取unique\_ptr](#r23使用make_unique来获取unique_ptr)
	- [R.24:使用weak\_ptr来避免shared\_ptr循环引用](#r24使用weak_ptr来避免shared_ptr循环引用)
	- [R.30:原则上用T\*或者T&，而不是用smart pointer来传递参数](#r30只有在需要表达生命期语意前提下才用智能指针传递参数)
	- [R.32 使用unique\_ptr\<Widget\>来表达一个函数对Widget的所有权](#r32-使用unique_ptr来表达一个函数对widget的所有权)
	- [R.33 使用unique\_ptr\<Widget\>&参数来表达函数reseats Widget](#r33-使用unique_ptr参数来表达函数reseats-widget)
	- [R.34 使用shared\_ptr\<T\>参数来表达共享所有权](#r34-使用shared_ptr参数来表达共享所有权)
	- [R.35 shared\_ptr\<T\>&参数用来表达可能reseat这个shared\_ptr](#r35-shared_ptr参数用来表达可能reseat这个shared_ptr)
	- [R.36 传递const shared\_ptr\<T\>&来表示有可能保留对象引用计数](#r36-传递const-shared_ptr来表示有可能保留对象引用计数)
	- [R.37 不要传参一个来自aliased的smart pointer的指针或引用](#r37-不要传参一个来自aliased的smart-pointer的指针或引用)
       
### R.1:使用资源句柄和RAII来管理资源

**原因** 为了避免泄漏以及简化管理资源，C++语言层面的构造、析构函数刚好对应资源获取、释放，比如fopen/fclose，
lock/unlock，new/delete。每当你要处理资源，比如获取、释放资源的函数调用时，应该把它封装到一个对象中，
构造函数获取资源，析构函数释放资源。

**Example, bad**
```cpp
void send(X* x, string_view destination)
{
    auto port = open_port(destination);
    my_mutex.lock();
    // ...
    send(port, x);
    // ...
    my_mutex.unlock();
    close_port(port);
    delete x;
}
```
在以上代码中，你必须记得要unlock，close\_port，delete，不管任何一条路径都要处理到，而且，一旦在 ... 中抛出异常，x就会泄漏，my\_mutex就还会锁住。

**Example**
```cpp
void send(unique_ptr<X> x, string_view destination)  // x owns the X
{
    Port port{destination};            // port owns the PortHandle
    lock_guard<mutex> guard{my_mutex}; // guard owns the lock
    // ...
    send(port, x);
    // ...
} // automatically unlocks my_mutex and deletes the pointer in x
```
现在所有的资源都自动清理，所有的路径上都执行一次，无法是否有异常，不仅如此，函数签名也表示它接管了指针所有权。
那Port是如何封装的？
```cpp
class Port {
    PortHandle port;
public:
    Port(string_view destination) : port{open_port(destination)} { }
    ~Port() { close_port(port); }
    operator PortHandle() { return port; }

    // port handles can't usually be cloned, so disable copying and assignment if necessary
    Port(const Port&) = delete;
    Port& operator=(const Port&) = delete;
};
```


### R.2:接口使用裸指针来代表单个对象

**原因** 数组最好由一个容器来表示，vector表示拥有所有权，span表示非所有权，它们拥有足够的信息来做范围检查。

**Example, bad**
```cpp
void f(int* p, int n)   // n is the number of elements in p[]
{
    // ...
    p[2] = 7;   // bad: subscript raw pointer
    // ...
}
```
编译器不会读注释，如果没有其他代码，那就不知道p到底拥有几个元素。建议用span替换。

**Example**
```cpp
void g(int* p, int fmt)   // print *p using format #fmt
{
    // ... uses *p and p[0] only ...
}
```

例外：C-style一般传递以0结尾的字符串指针，使用zstring而不是char\*，来表示依赖这个约定。
注意：当前很多用指向单元素的指针都可以用引用，但nullptr可以表示一个合法的指针，引用不行。

**Enforcement** 

* 所有非container,view,iterator的指针计算（包括++），都应标记指出，但是这对老代码会产生巨多false positives。
* 所有数组名被当作单一指针传递都应该被标记。


### R.3:T\*不拥有所有权

**原因** 如果是拥有所有权的指针都会被标识出来，这样就可以可靠且有效的删除所指向对象。

**Example**
```cpp
void f()
{
    int* p1 = new int{7};           // bad: raw owning pointer
    auto p2 = make_unique<int>(7);  // OK: the int is owned by a unique pointer
    // ...
}
```
unique\_ptr保证资源会析构，不会泄漏，即使抛出异常。T\*不会。

**Example**
```cpp
template<typename T>
class X {
public:
    T* p;   // bad: it is unclear whether p is owning or not
    T* q;   // bad: it is unclear whether q is owning or not
    // ...
};
```
优化
```cpp
template<typename T>
class X2 {
public:
    owner<T*> p;  // OK: p is owning
    T* q;         // OK: q is not owning
    // ...
};
```

**Example, bad**
```cpp
Gadget* make_gadget(int n)
{
    auto p = new Gadget{n};
    // ...
    return p;
}

void caller(int n)
{
    auto p = make_gadget(n);   // remember to delete p
    // ...
    delete p;
}
```
返回一个指针意味着调用者不确定这个指针的生命期，谁来负责删除所指向对象呢？
如果Gadget move函数开销不大，比如该对象很小，或者移动该对象很高效，那么直接返回value
就可以。
```cpp
Gadget make_gadget(int n)
{
    Gadget g{n};
    // ...
    return g;
}
```
注意：本规则适用于factory模式。如果确实需要返回指针，比如指向子类的基类指针，那么应该返回smart pointer。

**Enforcement**

- delete一个非所有权指针要给出警告
- 在每条代码路径上都没能reset或者显式delete都应该给出警告
- new出来的对象被一个裸指针指向给出警告
- 函数内构造的对象被当作函数返回值，并且该对象有移动构造，那么应该建议直接返回value。


### R.4:T&不拥有所有权

**Example**
```cpp
void f()
{
    int& r = *new int{7};  // bad: raw owning reference
    // ...
    delete &r;             // bad: violated the rule against deleting raw pointers
}
```


### R.5:优先使用scoped对象，非必要不使用堆分配

**原因** 一个scoped对象指的是一个局部、全局对象或者成员，这意味着不用再有额外的allocation
和deallocation，scoped对象的成员也是个scoped对象，scoped对象的生命期管理由构造、析构函数管理。

**Example**
以下代码不够高效，因为由非必要的分配和释放内存操作，而且一旦在...部分抛出异常，会造成内存泄漏，
而且代码也不简洁。
```cpp
void f(int n)
{
    auto p = new Gadget{n};
    // ...
    delete p;
}
```
Instead
```cpp
void f(int n)
{
    Gadget g{n};
    // ...
}
```

**Enforcement**

- 在函数内，如果对象被分配，然后在各个代码路径上被释放，那么首选stack object。
- 如果unique\_ptr对象没有被移动，shared\_ptr没有被拷贝，reassigned or reset，那么应当发出警告。


### R.6:避免使用非const全局变量

**Example**
```cpp
struct Data {
    // ... lots of stuff ...
} data;            // non-const data

void compute()     // don't
{
    // ... use data ...
}

void output()     // don't
{
    // ... use data ...
}
```
还有谁可以修改data？

一个线程读取全局数据，另一个线程修改这个全局数据，就会data race。


### R.10:避免使用malloc()和free()

**原因** malloc()和free()不支持构造和析构，跟new和delete也不能混着用。

**Example**
```cpp
class Record {
    int id;
    string name;
    // ...
};

void use()
{
    // p1 might be nullptr
    // *p1 is not initialized; in particular,
    // that string isn't a string, but a string-sized bag of bits
    Record* p1 = static_cast<Record*>(malloc(sizeof(Record)));

    auto p2 = new Record;

    // unless an exception is thrown, *p2 is default initialized
    auto p3 = new(nothrow) Record;
    // p3 might be nullptr; if not, *p3 is default initialized

    // ...

    delete p1;    // error: cannot delete object allocated by malloc()
    free(p2);    // error: cannot free() object allocated by new
}
```

### R.11:避免显式调用new和delete

### R.12:资源的分配应立即丢给一个可管理对象

**原因** 不这样做，一个异常或返回都会导致泄漏

**Example,bad**
```cpp
void func(const string& name)
{
    FILE* f = fopen(name, "r");            // open the file
    vector<char> buf(1024);
    auto _ = finally([f] { fclose(f); });  // remember to close the file
    // ...
}
```
**Example**
```cpp
void func(const string& name)
{
    ifstream f{name};   // open the file
    vector<char> buf(1024);
    // ...
}
```

### R.13:在一个表达式中最多执行一次显式资源分配

**原因** 如果在一个表达式中有两次以上资源分配，因为子表达式的计算顺序比如函数参数，会导致内存泄漏。

**Example**
```cpp
void fun(shared_ptr<Widget> sp1, shared_ptr<Widget> sp2);
```
```cpp
// BAD: potential leak
fun(shared_ptr<Widget>(new Widget(a, b)), shared_ptr<Widget>(new Widget(c, d)));
```
编译器会重组这两个表达式，它不是异常安全的，有可能编译器先分配两段内存，然后再执行Widget构造函数，
如果其中一个构造函数抛出异常，那么就会内存泄漏。
一个简化的解决方案就是不要在一个表达式中有超过一次显式的资源分配。
```cpp
shared_ptr<Widget> sp1(new Widget(a, b)); // Better, but messy
fun(sp1, new Widget(c, d));
```
```cpp
fun(make_shared<Widget>(a, b), make_shared<Widget>(c, d)); // Best
```


### R.14:避免使用[]，用span

**原因** 数组会退化成指针，因此会丢失其size，造成range错误，使用span，它包含size信息。

**Example**
```cpp
void f(int[]);          // not recommended

void f(int*);           // not recommended for multiple objects
                        // (a pointer should point to a single object, do not subscript)

void f(gsl::span<int>); // good, recommended
```


### R.15:重载allocation/deallocation要配套

**Example**
```cpp
class X {
    // ...
    void* operator new(size_t s);
    void operator delete(void*);
    // ...
};
```
注意：如果希望内存不被释放，释放操作应该=delete。


### R.20:用unique\_ptr和shared\_ptr来表达所有权

**原因** 避免资源泄漏

**Example**
```cpp
void f()
{
    X x;
    X* p1 { new X };              // see also ???
    unique_ptr<X> p2 { new X };   // unique ownership; see also ???
    shared_ptr<X> p3 { new X };   // shared ownership; see also ???
    auto p4 = make_unique<X>();   // unique_ownership, preferable to the explicit use "new"
    auto p5 = make_shared<X>();   // shared ownership, preferable to the explicit use "new"
}
```


### R.21:优先使用unique\_ptr，除非要共享所有权才用shared\_ptr

**Example,bad**
```cpp
void f()
{
    shared_ptr<Base> base = make_shared<Derived>();
    // use base locally, without copying it -- refcount never exceeds 1
} // destroy base
```
**Example**
```cpp
void f()
{
    unique_ptr<Base> base = make_unique<Derived>();
    // use base locally
} // destroy base
```

**Enforcement**

- 如果函数内使用shared\_ptr分配对象，但函数没有返回shared\_ptr，而且没有把shared\_ptr传递给其他函数，那么建议用unique\_ptr替换。


### R.22:使用make\_shared来获取shared\_ptr

**Example**
```cpp
shared_ptr<X> p1 { new X{2} }; // bad
auto p = make_shared<X>(2);    // good
```

make\_shared更简洁，同时避免了对引用计数的单独allocation，它直接把引用计数放在其对象旁边。


### R.23:使用make\_unique来获取unique\_ptr

**Example**
```cpp
unique_ptr<Foo> p {new Foo{7}};    // OK: but repetitive

auto q = make_unique<Foo>(7);      // Better: no repetition of Foo
```

简洁，而且在复杂的表达式中更能保证异常安全。


### R.24:使用weak\_ptr来避免shared\_ptr循环引用

**原因** shared\_ptr依赖引用计数，循环依赖的引用计数永远不会清零。

**Example**
```cpp
#include <memory>

class bar;

class foo {
public:
  explicit foo(const std::shared_ptr<bar>& forward_reference)
    : forward_reference_(forward_reference)
  { }
private:
  std::shared_ptr<bar> forward_reference_;
};

class bar {
public:
  explicit bar(const std::weak_ptr<foo>& back_reference)
    : back_reference_(back_reference)
  { }
  void do_something()
  {
    if (auto shared_back_reference = back_reference_.lock()) {
      // Use *shared_back_reference
    }
  }
private:
  std::weak_ptr<foo> back_reference_;
};
```

### R.30:原则上用T\*或者T&，而不是用smart pointer来传递参数

R.30 等同于本条，如果有显式表达所有权语意，那么应当传递smart pointer。

**原因** 如果有表达所有权的语意，比如传递或者共享所有权，才使用smart pointer来传递参数。
不涉及到生命期修改，应当用裸指针或引用传参。

传递smart pointer限制了函数调用方，一个函数如果需要一个widgt，就应当接受任何widget对象，
而不是生命期要智能管理的对象。

传递一个shared\_ptr意味着一个runtime开销。

**Example**
```cpp
// accepts any int*
void f(int*);

// can only accept ints for which you want to transfer ownership
void g(unique_ptr<int>);

// can only accept ints for which you are willing to share ownership
void g(shared_ptr<int>);

// doesn't change ownership, but requires a particular ownership of the caller
void h(const unique_ptr<int>&);

// accepts any int
void h(int&);
```

**Example, bad**
```
// callee
void f(shared_ptr<widget>& w)
{
    // ...
    use(*w); // only use of w -- the lifetime is not used at all
    // ...
};

// caller
shared_ptr<widget> my_widget = /* ... */;
f(my_widget);

widget stack_widget;
f(stack_widget); // error
```

**Example, good**
```cpp
// callee
void f(widget& w)
{
    // ...
    use(w);
    // ...
};

// caller
shared_ptr<widget> my_widget = /* ... */;
f(*my_widget);

widget stack_widget;
f(stack_widget); // ok -- now this works
```

函数参数自然是在函数调用生命周期内，因此极少有生命周期问题。而且dangling指针可以在静态检查检出。

**Enforement**

- 如果一个函数需要一个智能指针(重载->或\*)参数，该智能指针是可拷贝的，但函数只调用(\*,->或get())其中之一，
那建议只传递T\*或者T&。
- 传递一个智能指针(重载->或\*)当参数，该智能指针是copyable/movable，但是在函数调用中没有用到拷贝/移动，而且
也不修改智能指针所指向数据，也不会被传递给另一个可以这样做的函数。这意味着所有权语意没有用到，那么应当被标记
改为传递T\*或者T&。

### R.30:只有在需要表达生命期语意前提下，才用智能指针传递参数


### F.60: 当no argument也是一个选项时，优先使用T\*而不是T&

原因：指针可能为空nullptr，但引用不会，不存在空引用。有时用nullptr来代表no object，如果不是这样，用引用会让
代码更简单。

Example
```cpp
string zstring_to_string(zstring p) // zstring is a char*; that is a C-style string
{
    if (!p) return string{};    // p might be nullptr; remember to check
    return string{p};
}

void print(const vector<int>& r)
{
    // r refers to a vector<int>; no check needed
}
```


### R.32 使用unique\_ptr<Widget>来表达一个函数对Widget的所有权

**原因** 以这种方式使用unique\_ptr，既记录又执行了函数调用的所有权转移。 

**Example**
```cpp
void sink(unique_ptr<widget>); // takes ownership of the widget

void uses(widget*);            // just uses the widget
```

**Example, bad**
```cpp
void thinko(const unique_ptr<widget>&); // usually not what you want
```

**Enforcement**

- 如果一个函数传入lvalue左值unique\_ptr，但是没有赋值或reset()，那么应该考虑换成T\*或T&。
- 如果一个函数传入const unique\_ptr<T>引用，那应该考虑换成const T\*或const T&。


### R.33 使用unique\_ptr<Widget>&参数来表达函数reseats Widget

**原因** 使用这种方式，即记录又执行了函数调用的reseat作用。

**Example**
```cpp
void reseat(unique_ptr<widget>&); // "will" or "might" reseat pointer
```

**Example, bad**
```cpp
void thinko(const unique_ptr<widget>&); // usually not what you want
```

**Enforcement**

- 如果传入左值引用unique\_ptr<T>当函数参数，但是函数内没有调用赋值或reset，建议用T\*或者T&替换。
- 如果一个函数传入const unique\_ptr<T>，建议用const T\*或者const T&替换。


### R.34 使用shared\_ptr<T>参数来表达共享所有权

**原因** 显式表达共享所有权。

**Example, Good**
```cpp
class WidgetUser
{
public:
    // WidgetUser will share ownership of the widget
    explicit WidgetUser(std::shared_ptr<widget> w) noexcept:
        m_widget{std::move(w)} {}
    // ...
private:
    std::shared_ptr<widget> m_widget;
};
```

**Enforcement**

- 如果函数传递左值shared\_ptr<T>参数，但是调用函数内没有对它赋值或reset()，建议用T\*或T&替换。
- 如果函数传递const shared\_ptr<T>或const shared\_ptr<T>&，但是没有任何拷贝或者移动给另外一个shared\_ptr，
建议用T\*或T&替换。
- 如果函数传递shared\_ptr<T>右值，那么建议直接传递T。


### R.35 shared\_ptr<T>&参数用来表达可能reseat这个shared\_ptr

**原因** 显式表达函数可能reseat，reseat表示一个引用或一个智能指针指向一个不同的对象。

**Example, good**
```cpp
void ChangeWidget(std::shared_ptr<widget>& w)
{
    // This will change the callers widget
    w = std::make_shared<widget>(widget{});
}
```

**Enforcement**

- 如果一个函数传入左值引用shared\_ptr<T>，但是函数内没有赋值或reset()，建议用T\*或T&替换。
- 如果一个函数传入const shared\_ptr<T>，或const shared\_ptr<T>&，但是没有任何拷贝或者移动给另外一个
shared\_ptr<T>，建议用T\*或T&替换。
- 如果传入一个右值引用shared\_ptr<T>，那么建议直接传递T。


### R.36 传递const shared\_ptr<T>&来表示有可能保留对象引用计数 

**Example, good**
```cpp
void share(shared_ptr<widget>);            // share -- "will" retain refcount

void reseat(shared_ptr<widget>&);          // "might" reseat ptr

void may_share(const shared_ptr<widget>&); // "might" retain refcount
```


### R.37 不要传参一个来自aliased的smart pointer的指针或引用

**原因** 引用计数丢失或者悬空指针的最大的一个原因就是违反本条规则，函数调用传入指针或者引用应该在同个调用链上，
在调用链的顶端应该从一个智能指针获得裸指针或引用，并保持对象存在，并且保证智能指针在整个调用链上没有
reset或者被重新赋值。

注意：有时候会先本地获取一个智能指针的拷贝，然后保持对象存活，直到整个调用链。

**Example**
```cpp
// global (static or heap), or aliased local ...
shared_ptr<widget> g_p = ...;

void f(widget& w)
{
    g();
    use(w);  // A
}

void g()
{
    g_p = ...; // oops, if this was the last shared_ptr to that widget, destroys the widget
}
```
以下调用过不了code review
```cpp
void my_code()
{
    // BAD: passing pointer or reference obtained from a non-local smart pointer
    //      that could be inadvertently reset somewhere inside f or its callees
    f(*g_p);

    // BAD: same reason, just passing it as a "this" pointer
    g_p->func();
}
```
修复以上bug，可以先在开头获得一个本地拷贝，这样可以保持引用计数。
```cpp
void my_code()
{
    // cheap: 1 increment covers this entire function and all the call trees below us
    auto pin = g_p;

    // GOOD: passing pointer or reference obtained from a local unaliased smart pointer
    f(*pin);

    // GOOD: same reason
    pin->func();
}
```

**Enforcement**
- 如果从一个不是本地的或者是有别名的智能指针变量，获取的指针或引用，在函数中被使用，那应该发出警告。
如果这个智能指针是shared\_ptr，那应该先获取该shared\_ptr的本地拷贝，然后再获取它的指针或引用。
